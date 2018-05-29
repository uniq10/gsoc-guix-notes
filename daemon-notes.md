## nix-daemon.cc

* **run()**

    Call `daemonLoop()`.

* **daemonLoop()**

    Wait for connection on sockets (given as parameters) and on connection call
    `acceptConnection()`.

* **acceptConnection()**

    Accept an incoming connection. If connection from UNIX socket, i.e., from
    the same system, set client pid, uid, gid appropriately, else, set them to
    -1. Set `trusted` to `true` if client is root user.

    Then fork a process (which does not die with parent) to handle the
    connection and call [`setsid()`](
    http://man7.org/linux/man-pages/man2/setsid.2.html) to run independently
    in background. Save the client pid, uid, gid and the connection fd to `from`
    and `to` and then call `processConnection()`.

* **processConnection()**

    Do intial exchange of MAGIC ints and exchange protocol information. If
    EUID is 0 (root) then ensure we are using `build-users-group`.

    Then open the store and read the operation required from client and call
    `performOp`.

* **performOp()**

    * **wopBuildPaths:** Call `store->buildPaths()` with list of derivations
        and mode. Mode can only be `bmRepair` if client is `trusted`, else it
        is `bmNormal` (by default) or `bmCheck`.

        `buildPaths()` is defined in `nix\libstore\build.cc`.

## misc.cc

* **computeFSClosure()**

    It takes three arguments, a `store` object, a `path`, a list of paths
    `paths` and  bools `flipDirection` (default: `false`), `includeOutputs`
    (default: `false`) and `includeDerivers` (default: `false`).

    By default `flipDirection` (`false`) the functions adds to `paths` all paths
    that can be directly or indirectly reached from `path` using the references
    relation. "A" references "B" if "B" is needed while building "A". Therefore
    if `path` is "A" then "B" gets added to `paths` and also everything which
    "B" references will also be added to `paths`.

    For `flipDirection == true` the referrers relation is used. "A" is a
    referrer of "B" if "A" is needed while building "B".

    `includeOutputs` when `true` will also add outputPaths of derivations
    (which are valid and present in store) ecountered during closure when
    `flipDirection == false`. When `flipDirection` is `true` `includeOutputs`
    will the derivation paths (.drv files)  which are valid and encountered
    during the closure.

    `includeDerivers` when `true` and when `flipDirection == true` will add
    derivations paths(.drv files) encountered during the closure to `paths`.
    When `flipDirection == false` it will add the valid outputPaths of all
    derivations encountered during the closure.

    NOTE: The original `path` argument is always added to `paths`.


## build.cc

* **Goal**

    This is an abstract class representing the work to be done (Building
    derivation, substituion, etc.). It is initialized with a link to a `Worker`.

    It maintains a set of goals it is waiting for (`waitees`) and a set of
    goals waiting on it (`waiters`) and statistics on the `waitees`.


    * **nrFailed :** Number of Goals waiting for that failed.
    * **nrNoSubstituters :** Number of SubstituionGoals waiting for that
            failed because they had no substituters.

    * **nrIncompleteClosure :** Number of SubstituionGoals waiting for that
            failed beacuse they had unsubstitutable references.

    * **Goal::addWaitee()**

        Adds a given Goal to the `waitee` set. Also adds the Goal object to the
        `waiters` set of the given Goal.

    * **Goal::waiteeDone()**

        Takes a Goal in `waitee` as argument and removes it from the `waitee`
        set. It also updates the statistics on `waitees` appropriately.

        If after removal `waitee` set empty or if the argument Goal failed
        and `settings.keepGoing` is not set, remove all remaining Goals in
        `waitee` and remove the current Goal from the `waiters` set of remaining
        Goals in `waitee`. Then it clears the `waitees` set and calls
        `worker.wakeUp(this_goal)`.

    * **Goal::amDone()**

        Called when a Goal is done. This checks the `waiters` set for valid
        Goals waiting on it and calls `waiteeDone()` on them. Then it clears the
        `waiters` set and removes the Goal from `Worker`.

* **WeakGoalMap**

    This maps Paths to Goals.

* **DerivationGoal()**

    This class inherits from `Goal` class and deals with building derivations.
    It is initialized with a Path to a derivation (`drvPath`), set of output
    paths wanted (`wantedOutputs`), a `worker` to which the Goal will be added
    and the build mode (`buildMode`).

    If `wantedOutputs` is empty it means that all outputs specified by the
    derivation are to be built. The constructor sets the Goal state to
    `DerivationGoal::init` and adds the `drvPath` to the `store` tempRoot so
    that it won't be garbage collected. Calling `store.addTempRoot(drvPath)`
    ensures that `drvPath` will be `true` on `isActiceTempFile()` during
    garbage collection.

    **NOTE:** The `tempRootFile` stored is associated with the pid of the
    process using the store. Therefore, after a build is done and the process
    exits the file can be successfully garbage collected. This is accomplished
    by holding a read-lock on the process's tempRootFile during `addTempRoot()`.
    This mechanism is the reason why there is no need for an explicit
    `removeTempRoot()`.

    * **inputPaths:** This refers to all input paths, which is the union of
    immediate input paths (`drv.inputDrvs` outputs required and `drv.inputSrcs`)
    and its FSClosure, required to build the DerivationGoal.

    * **DerivationGoal::killChild()**

        The Goal when ready for build is assigned a process to build by the
        `worker` and has an associated `pid`. This call asks the `worker` to
        kill that child process. This is called when the build times out and
        also by the destructor.

    * **DerivationGoal::work()**

        This just calls the function associated with the current `state` of the
        Goal.

    * **DerivationGoal::addWantedOutputs()**

        If the argument `outputs` is empty then we clear `wantedOutputs`
        indicating that we need all outputs. Any change in `wantedOutputs` will
        set the `needRestart` to `true`.

    * **DerivationGoal::init()**

        `init` is the first `state` of the Goal and this is the function
        associated with `init`.

        This checks if the `drvPath` exists and if it does changes the `state`
        to `haveDerivation`. If it doesn't it adds a new `SubstituionGoal` to
        the `worker` and make the DerivationGoal `waiter` of the
        SubstituionGoal. It then changes the `state` to `haveDerivation`. This
        change of state is okay because the `haveDerivation` function will only
        be run on a `DerivationGoal::work()` which will only happen if all the
        `waitees` of the Goal were built successfully. Therefore, when
        `haveDerivation` is run it's `drvPath` is guranteed to exist.

     * **DerivationGoal::haveDerivation()**

        This function first checks if any of the `waitees` failed (i.e., if
        `nrFailed != 0`), if so it calls
        `DerivationGoal::done(BuildResult::MiscFailure)`.

        Else it adds `drvPath` to tempRoot and then reads the derivation. It
        adds all the `DerivationOutputs` to tempRoot. It then checks if the
        output paths are valid as per the store with
        `DerivationGoal::checkPathValidity` and if all paths are valid the build
        is done and it calls `DerivationGoal::done(BuildResult::AlreadyValid)`.

        It then checks if any of the invalid outputs previously failed to build,
        because some other process may have tried and failed before we got the
        PathLock. This is done using `DerivationGoal::pathFailed()`. In case of
        path failure on any invalid output we call
        `DerivationGoal::done(BuildResult::CachedFailure)` and returns.

        If no output path failed, we first try to use Substitutes if the dameon
        settings and the derivation properties allow it. We do this by adding
        new `SubstituionGoal`s for output paths to the worker and make the
        DerivationGoal a `waiter` of the new SubstituionGoals. Then we set the
        state to `DerivationGoal::outputsSubstituted`.

        If no SubstituionGoals were added and if the Goal has no `waitee`s we
        immediately call `DerivationGoal::outputsSubstituted`.

    * **DerivationGoal::pathFailed()**

        Checks with the store if the argument path failed to build.
        If it faied to build call
        `DerivationGoal::done(BuildResult::CachedFailure)` and return true.

    * **DerivationGoal::done()**

        If `BuildResult` is success (i.e., `Built`, `Substituted` or
        `AlreadyValid`) it calls `Goal::amDone(ecSuccess)` else it
        calls with argument `ecFailed`.

        Else if `TimedOut` it sets `worker.timedOut` to `true` and if
        `PermanentFailure` or `CachedFailure` it sets `worker.permanentFailure`
        to `true`.

    * **DerivationGoal::outputsSubstituted()**

        This throws an error if any Goals the DerivationGoal was waiting on
        failed because of reasons other than "No substituters found" or "Had
        unsubstitutable references". The error is not thrown if
        `settings.tryFallback` is set. When it is set, we will attempt to build
        the derivation if substituion failed. Note that so far our
        DerivationGoal will only be waiting on SubstituionGoals. No
        DerivationGoal will have been added to the `waitees` till this
        function is called.

        In case a substituion failed due to unsubstitutable references we can
        build the dependencies of the derivation and can use the substitues for
        the original derivation. Therefore if that is the case we set
        `retrySubstituion` to `true`.

        It resets all the `waitees` stats and check is `needRestart` is set
        (This happens when new outputs are added to `wantedOutputs`). If it is
        set we call `DerivationGoal::haveDerivation()` again and returns.

        It then checks if all needed output paths are valid. If all of them are
        valid and the `buildMode` is  `bmNormal`
        `DerivationGoal::done(BuildResult::Substituted)` is called and returns.

        If not it means that inputs of the derivation must be built before we
        can build it. `wantedOutputs` is set to empty indicating all outputs are
        required (and needs to be valid). DerivationGoals are created for all
        `drv.inputDrv` and the current Goal is made to wait on them.
        SubstitutionGoals are also created for all `drv.inputSrcs` and the
        current Goal waits on them too.

        If the `waitees` list is empty after all this
        `DerivationGoal::inputsRealised()` is called immediately else the
        `state` is set to `DerivationGoal::inputsRealised`.

    * **DerivationGoal::inputsRealised()**

        This calls `DerivationGoal::done(BuildResult::DependencyFailed)` if any
        of the Goals it is waiting on failed and returns. So by this point
        all the input derivation the current DerivationGoal depends upon is
        fully built and the source files for the current DerivationGoal are also
        downloaded.

        If `retrySubstituion` was set it calls
        `DerivationGoal::haveDerivation()` again to try substituting the Goal
        again and returns.

        Now this will populate the `allPaths` property of the DerivationGoal.
        `allPaths` refers to the set of all referenceable paths, both input and
        output, required during the derivation build.

        First all the outputs of the derivation are added to `allPaths`. Then
        it iterates through the derivations which are needed as inputs to the
        target (`drv.inputDrvs`) and adds the FSClosure of all the required
        outputs specified in `drv.inputDrvs` to `inputPaths`.
        The FSClosure is computed with `includeOutputs=false` (this ensures that
        only the required outputs specified in `drv.inputDrvs` are added),
        `includeDerivers=false` and using the references relation (i.e., if "A"
        depends on "B", closure of "A" will add "B" and closure of "B"). The
        FSClosure of `drv.inputSrcs` are also added to `inputPaths` with the
        same parameters.

        After this the contents of `inputPaths` is appended to `allPaths`.

        The derivation is then marked a `fixedOutput` derivation if all of the
        `drv.outputs` has a non-null hash set. If the derivation is
        `fixedOutput` the `nrRounds` (which is the number of rounds to build
        the derivation for) is set to 1 since there is no point in repeating
        builds of `fixedOutput` derivations as they are already verified by
        their output hash.

        Then the `state` is set to `DerivationGoal::tryToBuild` and
        `worker.wakeUp(current_goal)` is called to indicate that the Goal is
        ready to work.

    * **DerivationGoal::tryToBuild**

        It first checks if some other Goal is the worker/process has obtained
        locks on the output paths and if so it means some Goal is working on
        getting those output paths. Therefore it waits for some Goal to finish
        by calling `worker.waitForAnyGoal()` and returns.

        If no one had obtained the output locks it tries to obtain locks on all
        output paths without waiting and if it cannot obtain any lock path, it
        calls `worker.waitForAWhile()` and returns.

        If it reaches this point it has obtained all output locks. If it
        previously had to `waitForAWhile()` or `waitForAnyGoal()` it meant that
        some other builder held the output path locks for build purposes. So it
        makes sense to check if all the requird `drv.outputs` are valid and if
        so the build is successful. In that case the output locks are deletd and
        `DerivationGoal::done(BuildResult::AlreadyValid)` is called and returns.

        Check again for any outputs that previously failed to build by other
        Goals which held the locks to output paths. If any of them failed, call
        `DerivationGoal::pathFailed()` and return.

        Delete all output paths which exists but are not valid and then add all
        non-valid output paths to `missingPaths`. Missing paths are the paths
        left to build.

        After that check if the build can be done locally, `buildLocally = true`
        if `buildMode != buildNormal` or if the `env.preferLocalBuild == true`
        and the build can be performed locally. A build can be performed locally
        if `drv.platform` is same as the machince platform (NOTE: `i686-linux`
        builds can be done on `x86_64-linux` too).

        If `buildLocally` is `false` look for build via build hooks. Else the
        build must be done locally. If a build slot is not available the Goal
        is made to wait for build slots by `worker.waitForBuildSlot()` and the
        output locks are released and returns.

        NOTE: If the build must be performed locally we proceed with the build
        even if `settings.maxBuildJobs` is `0`.

        If we decide to proceed with the build we call
        `DerivationGoal::startBuilder()` and then set state to
        `DerivationGoal::buildDone`. If there is a `BuildError` during
        `DerivationGoal::startBuilder()` all output locks are released, the
        `buildUser` is released and `worker.permanentFailure` is set to `true`.
        and `DerivationGoal::done(BuildResult::InputRejected)` is called.

    * **DerivationGoal::startBuilder()**




* **Worker**

    This is class with a set of top-level Goals it must complete. The class is
    initialized with a reference to the `store`.  It also maintains a map of
    additional Goals it must complete to build all the top-level Goals.

    Only a single worker is to be present to service an op.

    * **Worker::makeDerivationGoal()**



* **LocalStore::buildPaths()**

    Creates a `Worker` with the current store object as input, adds both
    derivation goals and substituion goals to the w

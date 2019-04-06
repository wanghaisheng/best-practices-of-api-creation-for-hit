
Build & Deploy Pipeline
=======================

The starter kit comes with a CircleCI [configuration file](https://docs.subzero.cloud/circleci/config.yml) that will build your code and push it to production.

mkdir .circleci
cd .circleci
curl -SLO https://docs.subzero.cloud/circleci/config.yml

*   [Signup](https://circleci.com/signup/) to CircleCI using your Github account.
*   Add your project's Github repo to the list repos CircleCI will build when new code is pushed.

Note

Once you add your project, CircleCI will immediately start building your project. This is ok, it will not push it to production yet. When adding new repo in Circle 2 it does not build successfully the first time and it gives a: job "build" is not found, this cannot be fixed through a GUI 'rebuild', the build will be successful only on a new commit [Issue #13549](https://discuss.circleci.com/t/workflows-job-build-not-found/13549)\*

*   In CircleCI interface, go to the configuration of your project and setup all the env variables listed at the top of the config file

With this last step, your production infrastructure is ready to receive the first version of your application.


Exploration 3 Automated Deployments
----------------------------
For exploration 3 I wanted to explore automated deployments for my Capstone Project Ease
using Travis CI.

Travis CI is a Continuous Integration platform that integrates with github to automatically
run unit tests and show the status of a build when new commits are pushed to a branch.

To configure TravisCI all one needs to do is add a .travis.yml file to the root of their 
github project. 
Mine can be found [here.](https://github.com/EaseApp/web-backend/blob/master/.travis.yml)
All the .travis.yml file does is tell the travis servers how to configure their environment to
install our application and run the unit tests.  This is primarily seen in the before_install
section of the file. After that is where the automated deployment code really occurs. 

First the private key is decrypted so that the travis server will be able to communicate with 
the Ease server using SSL. 

`- openssl aes-256-cbc -K $encrypted_cab47c9f148d_key -iv $encrypted_cab47c9f148d_iv -in deploy_key.pem.enc -out .travis/deploy_key.pem -d`

Next is the after_success section. This section only runs if all of the tests pass, ensuring 
that a broken build will not be deployed.  

The following events then occur in order. 
The ssh agent is started
`eval "$(ssh-agent -s)`

The key is private key is made readable
`chmod 600 .travis/deploy_key.pem`

the private key is added temporarily to the travis server
`ssh-add .travis/deploy_key.pem`

The location of the bare git repository is added to the travis server
`git remote add deploy felttrip@ease-62q56ueo.cloudapp.net:/home/felttrip/git/web-backend`

and finally the branch is pushed to the production server
`git push deploy`

As far as travis is concerned the job was a success and it logs it as such. 

Next we jump to the production server where the application is actually built and deployed. 

Here I used a really useful feature of bare git repositories which is the post-receive
hook. What this does is allow me to run a script when the bare repository has been pushed to, whcih is what the travis server just did. 

I haven't versioned this script because it is pretty short but i'll add it here so you can look at it. 
```
#!/bin/sh
GIT_REPO=/home/felttrip/git/web-backend
TMP_GIT_CLONE=/home/felttrip/go/src/github.com/EaseApp/web-backend
SERVER_LOCATION=/home/felttrip

git clone $GIT_REPO $TMP_GIT_CLONE
cd $TMP_GIT_CLONE
make build

supervisorctl stop web-backend
cp -rp $TMP_GIT_CLONE/bin/main $SERVER_LOCATION
supervisorctl start web-backend
rm -rf $TMP_GIT_CLONE
```
The short of it is that the freshly pushed code is first cloned into a temporary location.
After that make build is run to build the production executable. 
Next I stop supervisor, which is a process monitoring tool I installed. Go applications
don't have a good way of making them dameons to supervisor is used to ensure the application
stays running. 
After supervisor is stopped the freshly built server executable is copied to the actual server location.
Supervisor is then told to start the application with the new binary.
FInally the temporary cloning space is cleaned up.

The end result being that when changes are merged into master for my capstone project 
they will be automatically deployed for usage by the rest of the team. This is especially important 
because We have two separate teams for frontend and backend work so having an constantly up to date
backend is extremely useful.

Resources
------------
http://git-scm.com/book/en/v2/Git-on-the-Server-Getting-Git-on-a-Server
http://docs.travis-ci.com/user/deployment/custom/
http://supervisord.org/

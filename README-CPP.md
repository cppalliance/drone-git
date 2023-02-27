# CPPAL customizations

To build a customized s390x drone-git image:  

2025-03-17 For multiple years previously this image existed and was used: cppalliance/git:linux-s390x

Building a new image:  
docker build . -f docker/Dockerfile.linux.s390x -t cppalliance/git:linux-s390x.v2
docker push cppalliance/git:linux-s390x.v2

This requires docker/Dockerfile.linux.s390x to exist, and docker/manifest.tpl to include s390x.
Will open an upstream PR although they are not always responsive. 

Specify the clone image:
-e DRONE_RUNNER_CLONE_IMAGE=cppalliance/git:linux-s390x.v2


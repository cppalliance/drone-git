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

# Windows Server 2025 image

2026-08-13 Upstream only ships windows-1809 and windows-ltsc2022 images, which will not
run with process isolation on a Windows Server 2025 host. Note that
mcr.microsoft.com/powershell has no nanoserver-ltsc2025 tag, so PowerShell 7 is unzipped
from a GitHub release instead of being inherited from that base image.

2026-08-18 Added the profile.ps1 line in the Dockerfile below. Without it every clone
fails with "fatal: unable to read tree (sha)"; see the comment in the Dockerfile.

Run the following on a Windows Server 2025 machine (so the image matches the host), from an
elevated PowerShell prompt. The Dockerfile is written to $env:TEMP rather than to docker/
to keep the fork free of extra files.

The build must run from the root of a checkout, not from C:\ or a home directory: the "."
in the docker build command is the build context, and that is where "ADD windows/*" looks
for the clone scripts. Running it from the wrong directory gives
"ADD failed: no source files were specified".

```powershell

# The current dir should be approximately as follows, or a similar idea:
# git clone https://github.com/cppalliance/drone-git C:\drone-git
# cd C:\drone-git

Set-Content -Path "$env:TEMP\Dockerfile.windows.ltsc2025" -Value @'
# escape=`

FROM mcr.microsoft.com/windows/servercore:ltsc2025 AS git
SHELL ["powershell.exe", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

RUN [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12 ; `
    Invoke-WebRequest -UseBasicParsing https://github.com/git-for-windows/git/releases/download/v2.55.0.windows.4/MinGit-2.55.0.4-64-bit.zip -OutFile git.zip; `
    Expand-Archive git.zip -DestinationPath C:\git; `
    Invoke-WebRequest -UseBasicParsing https://github.com/PowerShell/PowerShell/releases/download/v7.4.18/PowerShell-7.4.18-win-x64.zip -OutFile pwsh.zip; `
    Expand-Archive pwsh.zip -DestinationPath C:\pwsh;

FROM mcr.microsoft.com/windows/nanoserver:ltsc2025
COPY --from=git /git /git
COPY --from=git ["/pwsh", "C:\\Program Files\\PowerShell"]

ADD windows/* /bin/

USER ContainerAdministrator
RUN setx /M PATH "%PATH%;C:\Program Files\PowerShell"

# The clone scripts pass an empty $FLAGS argument to git when PLUGIN_DEPTH is unset.
# PowerShell 7.2 (the old base image) silently dropped empty arguments; 7.3+ passes
# them through, so git runs "git fetch <empty> origin <refspec>", which treats the
# empty string as an empty remote group and fetches nothing (exit 0, no output). The
# checkout then fails with "unable to read tree". Restore the old argument behavior
# via the machine-wide profile, which pwsh loads for C:\bin\clone.ps1.
RUN echo $PSNativeCommandArgumentPassing = 'Legacy' > "C:\Program Files\PowerShell\profile.ps1"

CMD [ "pwsh", "C:\\bin\\clone.ps1" ]
'@

docker build . -f "$env:TEMP\Dockerfile.windows.ltsc2025" -t cppalliance/git:windows-ltsc2025-amd64
# run this also:
# docker login
# docker push cppalliance/git:windows-ltsc2025-amd64
```

Specify the clone image:
-e DRONE_RUNNER_CLONE_IMAGE=cppalliance/git:windows-ltsc2025-amd64


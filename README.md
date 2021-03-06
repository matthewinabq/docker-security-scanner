
# Example

Builds Docker image which invokes security scripts.

Requires Nexus CLI JAR file and Twistlock Scanner executable.  You can get these from those vendors.

Copy the files into packages.

Update twistlock.py with the correct Nexus CLI Jar filename if using Nexus

## Script Library

### twistlock.py

Executes Twistlock CLI to scan Docker image given.

You can remove all NEXUS environment variables from the command below and add `-e "TL_ONLY=TRUE"` and it will run just a Twistlock scan.

TTY into Docker VM
Mac:
```console
$ screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty
```

Run script from Docker VM
Mac:
```console
/ # docker run -v /var/run/docker.sock:/var/run/docker.sock -v /var/lib/docker:/var/lib/docker --privileged -e "NEXUS_IQ_APPLICATION_ID=<iq_app_id>" -e "NEXUS_IQ_USERNAME=<ad_username>" -e "NEXUS_IQ_PASSWORD=<ad_password>" -e "NEXUS_IQ_STAGE=<iq_stage>" -e "TL_CONSOLE_HOSTNAME=<twistlock console hostname>" -e "TL_CONSOLE_PORT=443" -e "TL_CONSOLE_USERNAME=<tl_username>" -e "TL_CONSOLE_PASSWORD=<tl_password>" -e "NEXUS_IQ_URL=<nexus_iq_url>" -e "JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64" -e "JAVA_KEYSTORE_PASSWORD=changeit" <repository>:<tag> twistlock.py -i <image_id>
```

JAVA_KEYSTORE_PASSWORD can be anything you want feel free to update.

Codefresh Build Step to execute Twistlock/Nexus scan
All `${{var}}` variables must be put into Codefresh Build Parameters
codefresh.yml
```console
  buildimage:
    type: build
    title: Build Runtime Image
    dockerfile: Dockerfile
    image_name: # Image you're building/scanning [repository/image]
    tag: latest-cf-build-candidate

  nexus_iq_scan_build_stage:
    type: composition
    composition:
      version: '2'
      services:
        imagebuild:
          image: ${{buildimage}}
          command: sh -c "exit 0"
          labels:
            build.image.id: ${{CF_BUILD_ID}}
    composition_candidates:
      scan_service:
        image: # Add your scanning image [repository/image:tag]
        environment:
          - NEXUS_IQ_URL=${{NEXUS_IQ_URL}}
          - NEXUS_IQ_APPLICATION_ID=${{NEXUS_IQ_APPLICATION_ID}}
          - NEXUS_IQ_USERNAME=${{NEXUS_IQ_USERNAME}}
          - NEXUS_IQ_PASSWORD=${{NEXUS_IQ_PASSWORD}}
          - NEXUS_IQ_STAGE=${{NEXUS_IQ_STAGE}}
          - TL_CONSOLE_HOSTNAME=${{TL_CONSOLE_HOSTNAME}}
          - TL_CONSOLE_PORT=${{TL_CONSOLE_PORT}}
          - TL_CONSOLE_USERNAME=${{TL_CONSOLE_USERNAME}}
          - TL_CONSOLE_PASSWORD=${{TL_CONSOLE_PASSWORD}}
          - JAVA_HOME=${{JAVA_HOME}}
          - JAVA_KEYSTORE_PASSWORD=${{JAVA_KEYSTORE_PASSWORD}}
        command: twistlock.py -i "$$(docker inspect $$(docker inspect $$(docker ps -aqf label=build.image.id=${{CF_BUILD_ID}}) -f {{.Config.Image}}) -f {{.Id}} | sed 's/sha256://g')"
        depends_on:
          - imagebuild
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          - /var/lib/docker:/var/lib/docker
```
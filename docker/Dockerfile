FROM registry.access.redhat.com/ubi9-minimal:latest

# Builds a composite docker service that the "owner" needs to run. Specifically it contains the following services:
#   Intel SDO development/test rendezvous service
#   Intel SDO to0scheduler service
#   Intel SDO OPS (Owner Protocol Service)
#   Intel SDO OCS (Owner Companion Service)
#   Open-horizon ocs-api (OCS REST API)

# Building a UBI image:
#   https://access.redhat.com/articles/4238681
#   https://playbook.cloudpaklab.ibm.com/ubi-faq/#Q_Can_I_addenable_additional_package_repositories_for_UBI_images

# docker build requirements:
#   Follow the Developer's section of the top-level README.md to download the necessary SDO tar files and build this docker image

# docker run requirements (see run-sdo-owner-services.sh):
#   Mount a named docker volume at WORKDIR/ocs/config/db
#   Pass in the env vars required by start-sdo-owner-services.sh
#   Expose the ports in the EXPOSE ports below

# Image size reduction todos (0.9.3 is 1.03GB):
# Following https://docs.docker.com/develop/dev-best-practices/ and others, e.g. https://www.ardanlabs.com/blog/2020/02/docker-images-part1-reducing-image-size.html
# (future) run all sdo services in same tomcat instance

# Note: if ENV variables are used in this Dockerfile, they can not effectively be overridden by docker run
# WORKDIR: the working dir for the 2nd half of this file, and what all of the services will run from, SDO_OCS_DB_PATH: the ocs service read this. SDO_OCS_API_PORT and VERBOSE can be overridden on docker run.
ENV WORKDIR=/home/sdouser
ENV SDO_OCS_DB_PATH=$WORKDIR/ocs/config/db
# These env vars are only used at runtime (by start-sdo-owner-services.sh), so can be overridden by docker run
ENV SDO_RV_PORT=8040   SDO_OPS_PORT=8042   SDO_OCS_API_PORT=9008   VERBOSE=true
ENV SDO_OPS_EXTERNAL_PORT=$SDO_OPS_PORT
ENV SDO_OCS_API_TLS_PORT=$SDO_OCS_API_PORT

# Working during during the installations
WORKDIR /root

# The haveged rpm (needed by RV) is in EPEL, so add that repo: https://fedoraproject.org/wiki/EPEL
# To see the available EPEL pkgs: http://mirror.nodesdirect.com/epel/8/Everything/x86_64/Packages/
# Note: supposedly we could get this from the RHEL repo, if we go thru the right paperwork: https://developers.redhat.com/blog/2020/02/26/red-hat-simplifies-container-dev-and-redistribution-rhel-packages/
# To search for rpms (or for what rpms provides a command): http://www.rpm-find.net/linux/RPM/
# Note: libssl-dev is is required by all the sdo services on ubuntu. I think the equivalent rpm is openssl-devel
# Note: due to a bug in microdnf, using the --nodocs option causes an exit code of 141: https://github.com/rpm-software-management/microdnf/issues/50
RUN mkdir -p /run/user/$UID && \
    microdnf update -y --nodocs 1>/dev/null 2>&1 && \
    microdnf install -y --nodocs ca-certificates findutils gettext java-17-openjdk openssl procps shadow-utils tar && \
    microdnf clean all

RUN useradd -r -u 1000 -g root sdouser \
    && mkdir /home/sdouser \
    && chown -R sdouser:root /home/sdouser

# Install openjdk-11-jre. The tar.gz file is 43 MB, unpacked is 124 MB
# Note: with SDO 1.7, it is necessary to use java 11.0.4 or earlier, because there is an issue with the SDO bouncycastle version and 11.0.5 and above
#RUN curl -sS -o openjre-11_linux-x64_bin.tar.gz https://cdn.azul.com/zulu/bin/zulu11.33.15-ca-jre11.0.4-linux_x64.tar.gz && \
#RUN curl -sS -o openjre-11_linux-x64_bin.tar.gz https://cdn.azul.com/zulu/bin/zulu11.43.55-ca-jre11.0.9.1-linux_x64.tar.gz && \
#    mv zulu11.*-linux_x64 /usr/lib/jvm/openjre-11-manual-installation && \
RUN curl -sSL -o openjre-11_linux-x64_bin.tar.gz https://github.com/AdoptOpenJDK/openjdk11-binaries/releases/download/jdk-11.0.10%2B9/OpenJDK11U-jre_x64_linux_hotspot_11.0.10_9.tar.gz && \
    sha256sum openjre-11_linux-x64_bin.tar.gz && \
    tar xzf openjre-11_linux-x64_bin.tar.gz && \
    rm openjre-11_linux-x64_bin.tar.gz && \
    mkdir -p /usr/lib/jvm && \
    mv jdk-11.0.10+9-jre /usr/lib/jvm/openjre-11-manual-installation && \
    update-alternatives --install /usr/bin/java java /usr/lib/jvm/openjre-11-manual-installation/bin/java 1

# this doesn't work here, because docker writes /etc/hosts when it starts the container
#RUN echo "127.0.0.1 RVSDO OwnerSDO" >> /etc/hosts

USER sdouser
WORKDIR $WORKDIR

# Get the license file
COPY LICENSE.txt /licenses/
COPY sdo/NOTICES-v1.10.7/iot-platform-sdk/* /licenses/SDOIotPlatformSDK/

# Get rendezvous files. The rv subdir will be created automatically by COPY
# Note: need to use uid and gid to be able to build on non-linux hosts
COPY --chown=1000:0 sdo/pri-v1.10.7/rendezvous/* $WORKDIR/rv/

# Get to0scheduler files
COPY --chown=1000:0 sdo/iot-platform-sdk-v1.10.7/to0scheduler $WORKDIR/to0scheduler/

# Get OCS files
# Note: we don't want the contents of ocs/config/db, that is excluded in the .dockerignore file
COPY --chown=1000:0 sdo/iot-platform-sdk-v1.10.7/ocs $WORKDIR/ocs/
# This is only the default owner private key. It will be moved into ocs/config/db/v1/creds by start-sdo-owner-servic es.sh if they don't specify their own key in run-sdo-owner-services.sh
COPY --chown=1000:0 keys/sample-owner-keystore.p12 $WORKDIR/ocs/config/sample-owner-keystore.p12
#This command gives `USER sdouser` the correct permissions for mounting this key inside the container
RUN touch ocs/config/owner-keystore.p12 && chown sdouser:root ocs/config/owner-keystore.p12
# SDO_OCS_DB_PATH is where the named volume will get mounted to. Create the subdirs, and make sure we own everything
RUN mkdir -p $SDO_OCS_DB_PATH/v1/{creds,devices,values} ocs-api-dir/keys && chown -R 1000:0 $SDO_OCS_DB_PATH ocs-api-dir

# Get OPS files
COPY --chown=1000:0 sdo/iot-platform-sdk-v1.10.7/ops $WORKDIR/ops/

# Get our ocs-api binary, startup script, agent-install-wrapper.sh, and keystore scripts
COPY --chown=1000:0 ocs-api/linux/ocs-api ocs-api/scripts/*.sh docker/start-sdo-owner-services.sh $WORKDIR/

# Note: the EXPOSE stmt doesn't actually expose the port, it just serves as documentation about the -p flags docker run should use. We may override these values, so just let docker run set them.
#EXPOSE 8040  8042  9008
# RV: 8040, OPS: 8042, Our ocs-api: 9008
# Note: the OCS and to0scheduler traffic is all internal to this container, so don't need to expose 9009 and 8049, respectively

# Pass db path and ocs-api port in as env vars, so they can be set via docker run if desired
CMD $WORKDIR/start-sdo-owner-services.sh
#CMD ["bash"]

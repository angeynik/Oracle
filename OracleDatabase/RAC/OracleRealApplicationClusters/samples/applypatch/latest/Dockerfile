# LICENSE UPL 1.0
#
# Copyright (c) 1982-2019 Oracle and/or its affiliates. All rights reserved.
#
# ORACLE DOCKERFILES PROJECT
# --------------------------
# This is the Dockerfile for a patched Oracle Database 21c Release 3
#
# REQUIREMETNS FOR THIS IMAGE
# ----------------------------------
# The oracle/rac-database:21.3.0 image has to exist
#
# HOW TO BUILD THIS IMAGE
# -----------------------
# Put the downloaded patch(es) into the sub folders patch/0NN
# Run:
#      $ docker build -f Dockerfile -t oracle/rac-database:21.3.0-<patch level> .
#
# hadolint global ignore=DL3006,DL3025
ARG BASE_RAC_IMAGE=oracle/database-rac:21.3.0

# Pull base image
# ---------------
FROM $BASE_RAC_IMAGE as builder
ARG RAC_SLIM_IMAGE

# Labels
# ------
LABEL "provider"="Oracle"                                            \
      "issues"="https://github.com/oracle/docker-images/issues"      \
      "maintainer"="paramdeep Saini <paramdeep.saini@oracle.com>"    \
      "volume.setup.location1"="/opt/scripts"                        \
      "volume.startup.location1"="/opt/scripts/startup"              \
      "port.listener"="1521"                                         \
      "port.oemexpress"="5500"

# Argument to control removal of components not needed after db software installation
ARG PATCH_DIR="patches"
ARG DB_EDITION="EE"
ARG USER="root"
ARG WORKDIR="/rac-work-dir"

# Environment variables required for this build (do NOT change)
# -------------------------------------------------------------
USER $USER

ENV PATCH_DIR=$PATCH_DIR \
    GRID_PATCH_FILE="applyGridPatches.sh" \
    DB_PATCH_FILE="applyDBPatches.sh" \
    FIXUP_PREQ_FILE="fixupPreq.sh" \
    DB_USER="oracle" \
    USER=$USER \
    WORKDIR=$WORKDIR \
    GRID_USER="grid"

# Use second ENV so that variable get substituted
ENV PATCH_INSTALL_DIR=/tmp/patches

# Copy binaries
# -------------
COPY $PATCH_DIR $PATCH_INSTALL_DIR/

# Install patches
RUN chown -R grid:oinstall $PATCH_INSTALL_DIR/*.sh && \
    chmod ug+x $PATCH_INSTALL_DIR/*.sh && \
    chmod ug+rwx $PATCH_INSTALL_DIR && \
    chown -R grid:oinstall $PATCH_INSTALL_DIR && \
    sed -e '/hard *memlock/s/^/#/g' -i /etc/security/limits.conf  && \
    sed -e '/ *nofile /s/^/#/g' -i /etc/security/limits.conf  && \
    sync && \
    su grid -c "$PATCH_INSTALL_DIR/$GRID_PATCH_FILE $PATCH_INSTALL_DIR" && \
    chown -R oracle:oinstall $PATCH_INSTALL_DIR

USER oracle

RUN $PATCH_INSTALL_DIR/$DB_PATCH_FILE $PATCH_INSTALL_DIR

USER $USER

RUN "$PATCH_INSTALL_DIR"/"$FIXUP_PREQ_FILE"  && \
    cp "$PATCH_INSTALL_DIR"/"$FIXUP_PREQ_FILE" "$SCRIPT_DIR"/"$FIXUP_PREQ_FILE" && \
    rm -rf /etc/oracle && \
    rm -rf "$PATCH_INSTALL_DIR"

#############################################
# -------------------------------------------
# Start new stage for grid/DB with Slim image
# -------------------------------------------
#############################################

FROM $RAC_SLIM_IMAGE as final
ARG USER

COPY --from=builder /u01 /u01
RUN mkdir -p /tmp/scripts
COPY --from=builder $SCRIPT_DIR /tmp/scripts
RUN cp -rn /tmp/scripts/* $SCRIPT_DIR/ && \
    rm -rf /tmp/scripts

RUN chmod 755 "$SCRIPT_DIR"/* && \
    "$INVENTORY"/orainstRoot.sh && \
    "$GRID_HOME"/root.sh && \
    "$DB_HOME"/root.sh && \
    "$SCRIPT_DIR"/"$FIXUP_PREQ_FILE" && \
    cp "$SCRIPT_DIR"/"$INITSH" /usr/bin/"$INITSH" && \
    chmod 755 /usr/bin/"$INITSH" && \
    rm -f "$SCRIPT_DIR"/"$FIXUP_PREQ_FILE"

USER $USER
WORKDIR $WORKDIR

VOLUME ["/common_scripts"]

# Define default command to start Oracle Grid and RAC Database setup.

ENTRYPOINT /usr/bin/$INITSH
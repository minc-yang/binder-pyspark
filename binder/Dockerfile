# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.
ARG OWNER=jupyter
ARG BASE_CONTAINER=$OWNER/scipy-notebook
FROM $BASE_CONTAINER

LABEL maintainer="Jupyter Project <jupyter@googlegroups.com>"

# Fix DL4006
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

# Spark dependencies
# Default values can be overridden at build time
# (ARGS are in lower case to distinguish them from ENV)
ARG spark_version="3.2.1"
ARG hadoop_version="3.2"
ARG spark_checksum="145ADACF189FECF05FBA3A69841D2804DD66546B11D14FC181AC49D89F3CB5E4FECD9B25F56F0AF767155419CD430838FB651992AEB37D3A6F91E7E009D1F9AE"
ARG openjdk_version="11"

ENV APACHE_SPARK_VERSION="${spark_version}" \
    HADOOP_VERSION="${hadoop_version}"

RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    "openjdk-${openjdk_version}-jre-headless" \
    ca-certificates-java && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Spark installation
WORKDIR /tmp
RUN wget -q "https://archive.apache.org/dist/spark/spark-${APACHE_SPARK_VERSION}/spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz" && \
    echo "${spark_checksum} *spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz" | sha512sum -c - && \
    tar xzf "spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz" -C /usr/local --owner root --group root --no-same-owner && \
    rm "spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz"

WORKDIR /usr/local

# Configure Spark
ENV SPARK_HOME=/usr/local/spark
ENV SPARK_OPTS="--driver-java-options=-Xms1024M --driver-java-options=-Xmx2048M --driver-java-options=-Dlog4j.logLevel=info" \
    PATH="${PATH}:${SPARK_HOME}/bin"

RUN ln -s "spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}" spark && \
    # Add a link in the before_notebook hook in order to source automatically PYTHONPATH
    mkdir -p /usr/local/bin/before-notebook.d && \
    ln -s "${SPARK_HOME}/sbin/spark-config.sh" /usr/local/bin/before-notebook.d/spark-config.sh


# Fix Spark installation for Java 11 and Apache Arrow library
# see: https://github.com/apache/spark/pull/27356, https://spark.apache.org/docs/latest/#downloading
RUN cp -p "${SPARK_HOME}/conf/spark-defaults.conf.template" "${SPARK_HOME}/conf/spark-defaults.conf" && \
    echo 'spark.driver.extraJavaOptions -Dio.netty.tryReflectionSetAccessible=true' >> "${SPARK_HOME}/conf/spark-defaults.conf" && \
    echo 'spark.executor.extraJavaOptions -Dio.netty.tryReflectionSetAccessible=true' >> "${SPARK_HOME}/conf/spark-defaults.conf"

# copy jars and config files
ADD binder/jars/stocator-1.1.4.jar /usr/local/spark-${spark_version}-bin-hadoop${hadoop_version}/jars/
ADD binder/jars/aws /usr/local/spark-${spark_version}-bin-hadoop${hadoop_version}/jars/
ADD binder/jars/azure /usr/local/spark-${spark_version}-bin-hadoop${hadoop_version}/jars/
ADD binder/jars/core-site.xml /usr/local/spark-${spark_version}-bin-hadoop${hadoop_version}/conf/

USER ${NB_UID}

# install pyspark
RUN pip install wget==3.2 pyspark==${spark_version} 

# jupyter lab extensions
RUN pip install jupyterlab-drawio
# disable these to reduce docker size
#RUN pip install jupyterlab-git
#RUN pip install Faker
# additional python libraries
# RUN pip install beautifulsoup4 
# RUN pip install seaborn 

# -- Layer: Stocator and COS SDK
# IBM Cloud
RUN pip install ibmos2spark
RUN pip install ibm-cos-sdk
RUN pip install matplotlib

# s3 browser
RUN pip install jupyterlab-s3-browser
# enable extensions
RUN jupyter serverextension enable --py jupyterlab_s3_browser
RUN jupyter labextension install jupyterlab-spreadsheet

# Install pyarrow
RUN mamba install --quiet --yes \
    'pyarrow=4.0.*' && \
    mamba clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

WORKDIR "${HOME}"

# copy notebooks and data
ADD binder/workspace/notebooks ${HOME}/
ADD binder/workspace/data ${HOME}/data
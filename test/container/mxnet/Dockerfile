FROM ubuntu:16.04

LABEL maintainer="Amazon AI"

# Specify accept-bind-to-port LABEL for inference pipelines to use SAGEMAKER_BIND_TO_PORT
# https://docs.aws.amazon.com/sagemaker/latest/dg/inference-pipeline-real-time.html
LABEL com.amazonaws.sagemaker.capabilities.accept-bind-to-port=true
# Specify multi-models LABEL to indicate container is capable of loading and serving multiple models concurrently
# https://docs.aws.amazon.com/sagemaker/latest/dg/build-multi-model-build-container.html
LABEL com.amazonaws.sagemaker.capabilities.multi-models=true

ARG MMS_VERSION=1.0.8
ARG MX_URL=https://aws-mxnet-pypi.s3-us-west-2.amazonaws.com/1.6.0/aws_mxnet_mkl-1.6.0rc0-py2.py3-none-manylinux1_x86_64.whl
ARG PYTHON=python3
ARG PYTHON_PIP=python3-pip
ARG PIP=pip3
ARG PYTHON_VERSION=3.6.8
ARG PIP_VERSION=19.3.1

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    LD_LIBRARY_PATH="${LD_LIBRARY_PATH}:/usr/local/lib" \
    PYTHONIOENCODING=UTF-8 \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    TEMP=/home/model-server/tmp

RUN apt-get update \
 && apt-get -y install --no-install-recommends \
    build-essential \
    ca-certificates \
    curl \
    git \
    libopencv-dev \
    openjdk-8-jdk-headless \
    vim \
    wget \
    zlib1g-dev \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*

RUN wget https://www.python.org/ftp/python/$PYTHON_VERSION/Python-$PYTHON_VERSION.tgz \
 && tar -xvf Python-$PYTHON_VERSION.tgz \
 && cd Python-$PYTHON_VERSION \
 && ./configure \
 && make \
 && make install \
 && apt-get update \
 && apt-get install -y --no-install-recommends \
    libreadline-gplv2-dev \
    libncursesw5-dev \
    libssl-dev \
    libsqlite3-dev \
    tk-dev \
    libgdbm-dev \
    libc6-dev \
    libbz2-dev \
 && make \
 && make install \
 && rm -rf ../Python-$PYTHON_VERSION* \
 && ln -s /usr/local/bin/pip3 /usr/bin/pip

RUN ln -s $(which ${PYTHON}) /usr/local/bin/python

RUN ${PIP} --no-cache-dir install --upgrade \
    pip==${PIP_VERSION} \
    setuptools

WORKDIR /

COPY mxnet/sagemaker_mxnet_serving_container.tar.gz /sagemaker_mxnet_serving_container.tar.gz

RUN ${PIP} install --no-cache-dir \
    ${MX_URL} \
    mxnet-model-server==$MMS_VERSION \
    keras-mxnet==2.2.4.1 \
    numpy==1.17.4 \
    onnx==1.4.1 \
    /sagemaker_mxnet_serving_container.tar.gz \
 && rm /sagemaker_mxnet_serving_container.tar.gz

# This is here to make our installed version of OpenCV work.
# https://stackoverflow.com/questions/29274638/opencv-libdc1394-error-failed-to-initialize-libdc1394
# TODO: Should we be installing OpenCV in our image like this? Is there another way we can fix this?
RUN ln -s /dev/null /dev/raw1394

##################################
# download models to model_store
##################################
RUN mkdir resnet_152 \
 && cd resnet_152 \
 && wget -O resnet-152-0000.params http://data.mxnet.io/models/imagenet/resnet/152-layers/resnet-152-0000.params \
 && wget -O resnet-152-symbol.json http://data.mxnet.io/models/imagenet/resnet/152-layers/resnet-152-symbol.json \
 && wget -O synset.txt http://data.mxnet.io/models/imagenet/synset.txt \
 && echo '[{"shape": [1, 3, 224, 224], "name": "data"}]' > resnet-152-shapes.json \
 && cd ..

RUN mkdir resnet_18 \
 && cd resnet_18 \
 && wget -O resnet-18-0000.params http://data.mxnet.io/models/imagenet/resnet/18-layers/resnet-18-0000.params \
 && wget -O resnet-18-symbol.json http://data.mxnet.io/models/imagenet/resnet/18-layers/resnet-18-symbol.json \
 && wget -O synset.txt http://data.mxnet.io/models/imagenet/synset.txt \
 && echo '[{"shape": [1, 3, 224, 224], "name": "data"}]' > resnet-18-shapes.json \
 && cd ..

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    LD_LIBRARY_PATH="${LD_LIBRARY_PATH}:/usr/local/lib" \
    PYTHONIOENCODING=UTF-8 \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

RUN useradd -m model-server \
    && mkdir -p /home/model-server/tmp \
    && chown -R model-server /home/model-server

#################################################
# custom_entrypoint uses custom_handler_service
#################################################
COPY mxnet/custom_handler.py /usr/local/bin/custom_handler.py
COPY mxnet/custom_entrypoint.py /usr/local/bin/dockerd-entrypoint.py
COPY mxnet/config.properties /home/model-server
COPY mxnet/deep_learning_container.py /usr/local/bin/deep_learning_container.py

RUN chmod +x /usr/local/bin/dockerd-entrypoint.py && \
    chmod +x /usr/local/bin/deep_learning_container.py

EXPOSE 8080 8081
ENV TEMP=/home/model-server/tmp
ENTRYPOINT ["python", "/usr/local/bin/dockerd-entrypoint.py"]
CMD ["mxnet-model-server", "--start", "--mms-config", "/home/model-server/config.properties"]

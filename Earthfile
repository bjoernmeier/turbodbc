os-base:
    ARG CODE_NAME=focal
    FROM ubuntu:$CODE_NAME

    ENV DEBIAN_FRONTEND=noninteractive
    # set locale to utf-8, which is required for some odbc drivers (mysql);
    # also, set environment as set after 'source /venv/bin/activate'
    ENV LC_ALL=C.UTF-8

    RUN apt-get update && apt-get upgrade -y && \
        apt-get install -y build-essential zlib1g-dev \
        wget unixodbc unixodbc-dev libboost-all-dev cmake g++ \
        odbc-postgresql postgresql-client ninja-build gnupg apt-transport-https && \
        apt-get clean

driver:
    ARG CODE_NAME=focal
    FROM --build-arg CODE_NAME="$CODE_NAME" +os-base

    # we need an mysql odbc driver for the integration tests
    # currently the test fail with a newer driver
    RUN cd /opt && \
        wget -q https://downloads.mysql.com/archives/get/p/10/file/mysql-connector-odbc-5.1.13-linux-glibc2.5-x86-64bit.tar.gz && \
        tar xzf mysql-connector-odbc-5.1.13-linux-glibc2.5-x86-64bit.tar.gz && \
        mysql-connector-odbc-5.1.13-linux-glibc2.5-x86-64bit/bin/myodbc-installer -d -a -n MySQL -t DRIVER=`ls /opt/mysql-*/lib/libmyodbc*w.so` && \
        rm /opt/*.tar.gz

    # we need an mssql odbc driver for the integration tests
    RUN wget -q https://packages.microsoft.com/keys/microsoft.asc -O- | apt-key add - && \
        wget -q https://packages.microsoft.com/config/debian/9/prod.list -O- > /etc/apt/sources.list.d/mssql-release.list && \
        apt-get update && \
        ACCEPT_EULA=Y apt-get install msodbcsql17 mssql-tools && \
        odbcinst -i -d -f /opt/microsoft/msodbcsql17/etc/odbcinst.ini

    # not used for the moment as Exasol is not tested here
    # RUN cd /opt && \
    #     wget -q https://www.exasol.com/support/secure/attachment/111075/EXASOL_ODBC-6.2.9.tar.gz && \
    #     tar xzf EXASOL_ODBC-6.2.9.tar.gz && \
    #     echo "\n[EXASOL]\nDriver=`ls /opt/EXA*/lib/linux/x86_64/libexaodbc-uo2214lv1.so`\nThreading=2\n" >> /etc/odbcinst.ini && \
    #     rm /opt/*.tar.gz

docker-cache:
    ARG CODE_NAME=focal
    FROM --build-arg CODE_NAME="$CODE_NAME" \
        +driver

    # hack to cache the docker setup and the image pull so later steps based on the same OS have this already included
    # this should be in sync with earth/docker-compose.yml
    WITH DOCKER --pull postgres:11 --pull mysql:5.6 --pull mcr.microsoft.com/mssql/server:2017-latest
        RUN echo ""
    END

python:
    ARG PYTHON_VERSION=3.8.6
    ARG CODE_NAME=focal
    ARG ARROW_VERSION_RULE="<2.0.0"
    ARG NUMPY_VERSION_RULE=""
    FROM --build-arg CODE_NAME="$CODE_NAME" +docker-cache

    ENV MINICONDA_URL="https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh"
    RUN cd /opt && \
        wget --no-verbose -O miniconda.sh $MINICONDA_URL && chmod +x miniconda.sh
    ENV MINICONDA=$HOME/miniconda
    RUN cd /opt && \
        ./miniconda.sh -b -p $MINICONDA && \
        . $MINICONDA/etc/profile.d/conda.sh && \
        conda config --remove channels defaults && \
        conda config --add channels conda-forge && \
        conda create -y -q -n turbodbc-dev \
        gcc_linux-64 \
        make \
        ninja \
        cmake \
        coveralls \
        gmock \
        gtest \
        gxx_linux-64 \
        mock \
        pytest \
        pytest-cov \
        python=${PYTHON_VERSION} \
        unixodbc \
        boost-cpp \
        numpy$NUMPY_VERSION_RULE \
        pyarrow$ARROW_VERSION_RULE \
        pybind11 \
        -c conda-forge

    RUN cd /opt && \
        wget -q https://github.com/pybind/pybind11/archive/v2.6.2.tar.gz && \
        tar xvf v2.6.2.tar.gz

build:
    ARG PYTHON_VERSION=3.8.6
    ARG CODE_NAME=focal
    ARG ARROW_VERSION_RULE="<2.0.0"
    ARG NUMPY_VERSION_RULE=""
    FROM --build-arg CODE_NAME="$CODE_NAME" \
        --build-arg PYTHON_VERSION="$PYTHON_VERSION" \
        --build-arg ARROW_VERSION_RULE="$ARROW_VERSION_RULE" \
        --build-arg NUMPY_VERSION_RULE="$NUMPY_VERSION_RULE" \
        +python

    COPY . /src

    ENV ODBCSYSINI=/src/earthly/odbc
    ENV TURBODBC_TEST_CONFIGURATION_FILES="query_fixtures_postgresql.json,query_fixtures_mssql.json,query_fixtures_mysql.json"

    RUN mkdir /src/build
    WORKDIR /src/build
    RUN ln -s /opt/pybind11-2.6.2 /src/pybind11
    RUN bash -c ". $MINICONDA/etc/profile.d/conda.sh && \
        conda activate turbodbc-dev && \
        export UNIXODBC_INCLUDE_DIR=$CONDA_PREFIX/include && \
        cmake -DBOOST_ROOT=$CONDA_PREFIX -DBUILD_COVERAGE=ON \
            -DCMAKE_INSTALL_PREFIX=./dist  \
            -DPYTHON_EXECUTABLE=/miniconda/envs/turbodbc-dev/bin/python \
            -GNinja .. && \
        ninja && \
        cmake --build . --target install"
    SAVE ARTIFACT /src/build /build

test:
    ARG PYTHON_VERSION=3.8.6
    ARG CODE_NAME=focal
    ARG ARROW_VERSION_RULE="<2.0.0"
    ARG NUMPY_VERSION_RULE=""
    FROM --build-arg CODE_NAME="$CODE_NAME" \
        --build-arg PYTHON_VERSION="$PYTHON_VERSION" \
        --build-arg ARROW_VERSION_RULE="$ARROW_VERSION_RULE" \
        --build-arg NUMPY_VERSION_RULE="$NUMPY_VERSION_RULE" \
        +build

    WITH DOCKER --compose ../earthly/docker-compose.yml
        RUN /bin/bash -c "\
            (r=20;while ! pg_isready --host=localhost --port=5432 --username=postgres ; do ((--r)) || exit; sleep 1 ;done) && \
            (r=5;while ! /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'StrongPassword1' -Q 'SELECT @@VERSION' ; do ((--r)) || exit; sleep 3 ;done) && \
            sleep 5 && \
            /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'StrongPassword1' -Q 'CREATE DATABASE test_db' && \
            . $MINICONDA/etc/profile.d/conda.sh && \
            conda activate turbodbc-dev && \
            ctest --verbose \
            "
    END

test-python3.6:
    ARG PYTHON_VERSION="3.6.12"
    ARG ARROW_VERSION_RULE="<2.0.0"

    BUILD --build-arg CODE_NAME="focal" \
        --build-arg PYTHON_VERSION="$PYTHON_VERSION" \
        --build-arg ARROW_VERSION_RULE="<2.0.0" \
        --build-arg NUMPY_VERSION_RULE="<1.20.0" \
        +test

test-python3.8-arrow0.x.x:
    ARG PYTHON_VERSION="3.8.5"
    BUILD --build-arg CODE_NAME="focal" \
        --build-arg PYTHON_VERSION="$PYTHON_VERSION" \
        --build-arg ARROW_VERSION_RULE="<1" \
        --build-arg NUMPY_VERSION_RULE="<1.20.0" \
        +test

test-python3.8-arrow1.x.x:
    ARG PYTHON_VERSION="3.8.5"
    BUILD --build-arg CODE_NAME="focal" \
        --build-arg PYTHON_VERSION="$PYTHON_VERSION" \
        --build-arg ARROW_VERSION_RULE=">1,<2" \
        --build-arg NUMPY_VERSION_RULE=">=1.20.0" \
        +test

test-python3.8-arrow2.x.x:
    ARG PYTHON_VERSION="3.8.5"
    BUILD --build-arg CODE_NAME="focal" \
        --build-arg PYTHON_VERSION="$PYTHON_VERSION" \
        --build-arg ARROW_VERSION_RULE=">2,<3" \
        --build-arg NUMPY_VERSION_RULE=">=1.20.0" \
        +test

test-python3.8-arrow3.x.x:
    ARG PYTHON_VERSION="3.8.5"
    BUILD --build-arg CODE_NAME="focal" \
        --build-arg PYTHON_VERSION="$PYTHON_VERSION" \
        --build-arg ARROW_VERSION_RULE=">3" \
        --build-arg NUMPY_VERSION_RULE=">=1.20.0" \
        +test

test-python3.8-all:
    BUILD test-python3.8-arrow0.x.x
    BUILD test-python3.8-arrow1.x.x
    BUILD test-python3.8-arrow2.x.x
    BUILD test-python3.8-arrow3.x.x

test-all:
    BUILD +test-python3.6
    BUILD +test-python3.8-all

docker:
    ARG PYTHON_VERSION=3.8.6
    ARG CODE_NAME=focal
    ARG ARROW_VERSION_RULE="<2.0.0"
    ARG NUMPY_VERSION_RULE="<1.20.0"

    FROM --build-arg CODE_NAME="$CODE_NAME" \
        --build-arg PYTHON_VERSION="$PYTHON_VERSION" \
        --build-arg ARROW_VERSION_RULE="$ARROW_VERSION_RULE" \
        --build-arg NUMPY_VERSION_RULE="$NUMPY_VERSION_RULE" \
        +build

    ENTRYPOINT ["/bin/bash"]
    SAVE IMAGE turbodbc:latest
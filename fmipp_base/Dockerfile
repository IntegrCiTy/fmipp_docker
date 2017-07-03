FROM buildpack-deps:jessie


################################################################################
#################     PYTHON INSTALLATION FROM python:3.5     ##################
################################################################################


# Ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH

# http://bugs.python.org/issue19846
# At the moment, setting "LANG=C" on a Linux system *fundamentally breaks Python 3*, and that's not OK.
ENV LANG C.UTF-8

# Runtime dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
	tcl \
	tk \
	&& rm -rf /var/lib/apt/lists/*

ENV GPG_KEY 97FC712E4C024BBEA48A61ED3A5CA953F73C700D
ENV PYTHON_VERSION 3.5.2

RUN set -ex \
	&& buildDeps=' \
		tcl-dev \
		tk-dev \
	' \
	&& apt-get update && apt-get install -y $buildDeps --no-install-recommends && rm -rf /var/lib/apt/lists/* \
	\
	&& wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz" \
	&& wget -O python.tar.xz.asc "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz.asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$GPG_KEY" \
	&& gpg --batch --verify python.tar.xz.asc python.tar.xz \
	&& rm -r "$GNUPGHOME" python.tar.xz.asc \
	&& mkdir -p /usr/src/python \
	&& tar -xJC /usr/src/python --strip-components=1 -f python.tar.xz \
	&& rm python.tar.xz \
	\
	&& cd /usr/src/python \
	&& ./configure \
		--enable-loadable-sqlite-extensions \
		--enable-shared \
		--without-ensurepip \
	&& make -j "$(nproc)" \
	&& make install \
	&& ldconfig \
	\
	&& apt-get purge -y --auto-remove $buildDeps \
	\
	&& find /usr/local -depth \
		\( \
			\( -type d -a -name test -o -name tests \) \
			-o \
			\( -type f -a -name '*.pyc' -o -name '*.pyo' \) \
		\) -exec rm -rf '{}' + \
	&& rm -rf /usr/src/python

# Make some useful symlinks that are expected to exist
RUN cd /usr/local/bin \
	&& ln -s idle3 idle \
	&& ln -s pydoc3 pydoc \
	&& ln -s python3 python \
	&& ln -s python3-config python-config

RUN pip --version; \
	\
	find /usr/local -depth \
		\( \
			\( -type d -a -name test -o -name tests \) \
			-o \
			\( -type f -a -name '*.pyc' -o -name '*.pyo' \) \
		\) -exec rm -rf '{}' +; \
	rm -f get-pip.py


################################################################################
###################     JAVA INSTALLATION FROM openjdk:7     ###################
################################################################################


# A few problems with compiling Java from source:
#  1. Oracle. Licensing prevents us from redistributing the official JDK.
#  2. Compiling OpenJDK also requires the JDK to be installed, and it gets
#       really hairy.

RUN apt-get update && apt-get install -y --no-install-recommends \
		bzip2 \
		unzip \
		xz-utils \
	&& rm -rf /var/lib/apt/lists/*

RUN echo 'deb http://deb.debian.org/debian experimental main' > /etc/apt/sources.list.d/experimental.list

# Default to UTF-8 file.encoding
ENV LANG C.UTF-8

# Add a simple script that can auto-detect the appropriate JAVA_HOME value
# Based on whether the JDK or only the JRE is installed
RUN { \
		echo '#!/bin/sh'; \
		echo 'set -e'; \
		echo; \
		echo 'dirname "$(dirname "$(readlink -f "$(which javac || which java)")")"'; \
	} > /usr/local/bin/docker-java-home \
	&& chmod +x /usr/local/bin/docker-java-home

# Do some fancy footwork to create a JAVA_HOME that's cross-architecture-safe
RUN ln -svT "/usr/lib/jvm/java-7-openjdk-$(dpkg --print-architecture)" /docker-java-home
ENV JAVA_HOME /docker-java-home

RUN set -ex; \
	\
	apt-get update && apt-get install -y --no-install-recommends \
		openjdk-7-jdk \
	; \
	rm -rf /var/lib/apt/lists/*; \
	\
# Verify that "docker-java-home" returns what we expect
	[ "$(readlink -f "$JAVA_HOME")" = "$(docker-java-home)" ]; \
	\
# Update-alternatives so that future installs of other OpenJDK versions don't change /usr/bin/java
	update-alternatives --get-selections | awk -v home="$(readlink -f "$JAVA_HOME")" 'index($3, home) == 1 { $2 = "manual"; print | "update-alternatives --set-selections" }'; \
# ... and verify that it actually worked for one of the alternatives we care about
	update-alternatives --query java | grep -q 'Status: manual'


################################################################################
########################     INSTALL BOOST LIBRARY     #########################
################################################################################

ENV BOOST_VERSION '1.63.0'
ENV BOOST_UNDER_VERSION '1_63_0'

RUN wget -q https://sourceforge.net/projects/boost/files/boost/${BOOST_VERSION}/boost_${BOOST_UNDER_VERSION}.tar.bz2

# Unzip and remove archive
RUN tar --bzip2 -xf boost_${BOOST_UNDER_VERSION}.tar.bz2 \
	&& rm -r boost_${BOOST_UNDER_VERSION}.tar.bz2

# Build and install boost - compile only needed libraries
RUN cd /boost_${BOOST_UNDER_VERSION} \
	&& ./bootstrap.sh --with-libraries=date_time,system,filesystem,test \
	&& ./b2 \
	&& ./b2 install \
	&& rm -r /boost_${BOOST_UNDER_VERSION}


################################################################################
#########################     INSTALL CMAKE 3.8.1     ##########################
################################################################################


ENV CMAKE_VERSION '3.8.1'

RUN wget -q https://cmake.org/files/v3.8/cmake-${CMAKE_VERSION}.tar.gz

# Unzip and remove archive
RUN tar -xf cmake-${CMAKE_VERSION}.tar.gz \
	&& rm -r cmake-${CMAKE_VERSION}.tar.gz

# Build and install cmake
RUN cd /cmake-${CMAKE_VERSION} \
	&& ./bootstrap \
	&& make \
	&& make install \
	&& rm -r /cmake-${CMAKE_VERSION}


################################################################################
########################     INSTALL FMIPP LIBRARY     #########################
################################################################################


# Install swig
RUN apt-get update && apt-get install -y swig

# Get fmipp code
RUN git clone https://git.code.sf.net/p/fmipp/code fmipp-code

# Build and install fmipp
RUN cd /fmipp-code \
	&& mkdir build \
	&& cd build \
	&& cmake .. \
	&& make \
	&& make install \
	&& rm -r /fmipp-code


################################################################################
#################################     CMD     ##################################
################################################################################


CMD ["/bin/bash"]

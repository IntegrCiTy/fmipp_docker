FROM ubuntu:16.04

RUN apt-get -y update \
    && apt-get -y -q install g++ subversion gfortran ipython cmake swig ant python-dev \
    cython python-lxml python-nose python-jpype \
    zlib1g-dev libboost-dev openjdk-8-jdk jcc python-numpy python-scipy python-matplotlib\
    && rm -rf /var/lib/apt/lists/*

RUN mkdir /home/jmodelica

RUN svn co --ignore-externals https://svn.jmodelica.org/trunk /build/jmodelica
WORKDIR /build/jmodelica

RUN svn propset svn:externals "$(svn propget svn:externals . | grep -v 'assimulo')" .
RUN svn up .
RUN svn revert .
WORKDIR /build/jmodelica/external
RUN svn co https://svn.jmodelica.org/assimulo/trunk Assimulo

RUN mkdir /build/jmodelica/build
WORKDIR /build/jmodelica/build
RUN ../configure --prefix=/home/jmodelica --without-ipopt 

RUN JAVA_TOOL_OPTIONS=-Dfile.encoding=UTF8 make install 

RUN rm -fR /build/jmodelica

WORKDIR /home

RUN mkdir /home/modelica-libs
ENV MODELICAPATH /home/jmodelica/ThirdParty/MSL:/home/modelica-libs

CMD ["/bin/bash"]

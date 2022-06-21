FROM ubuntu:20.04

RUN rm /etc/apt/sources.list
RUN echo deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse > /etc/apt/sources.list
RUN echo deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse >> /etc/apt/sources.list
RUN echo deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse >> /etc/apt/sources.list
RUN echo deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse >> /etc/apt/sources.list
RUN apt update
RUN apt install -y gcc-riscv64-unknown-elf
RUN apt install -y qemu-system-misc
RUN apt install -y python3
RUN apt install -y make
RUN apt install -y python3-pip
RUN apt install -y dosfstools
RUN python3 -m pip install pexpect

RUN mkdir /app

WORKDIR /app

#CMD /bin/bash -c 'qemu-system-riscv64 --help'
#CMD make run
CMD rm -f fs.img && make fs && python3 tester.py

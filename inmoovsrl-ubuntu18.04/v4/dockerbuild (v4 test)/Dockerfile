FROM nvidia/cudagl:10.0-devel-ubuntu18.04
LABEL maintainer "SJTU-lxd"

# docker build . -t inmoovsrl:v4.0
# docker run -it --runtime=nvidia -v /tmp/.X11-unix:/tmp/.X11-unix -v /data:/data -v /etc/localtime:/etc/localtime -v /code:/root/project -e DISPLAY=unix$DISPLAY -p 50001:22 -p 27017:27017 --name ROS1 inmoovsrl:v4.0
# python -m rl_baselines.train --env OmnirobotEnv-v0 --algo ppo2 --log-dir logs/test/ --srl-model ground_truth --no-vis
# python -m replay.enjoy_baselines --log-dir logs/dockertest/OmnirobotEnv-v0/ground_truth/ppo2/19-12-25_02h12_39/ --render --action-proba

#Installs dependencies.
RUN export DEBIAN_FRONTEND=noninteractive && apt-get update && apt-get install -y \
        sudo glmark2 mesa-utils \
	libopenmpi-dev openmpi-bin openmpi-doc cmake libz-dev \
	wget bzip2 ca-certificates \
	libglib2.0-0 libxext6 libsm6 libxrender1 \
	git mercurial subversion \
	swig && \
    rm -rf /var/lib/apt/lists/*

#Installs Anaconda
ENV PATH /usr/local/anaconda3/bin:$PATH
RUN wget --quiet https://repo.anaconda.com/archive/Anaconda3-5.3.0-Linux-x86_64.sh -O ~/anaconda.sh && \
    /bin/bash ~/anaconda.sh -b -p /usr/local/anaconda3 && \
    rm ~/anaconda.sh && \
    ln -s /usr/local/anaconda3/etc/profile.d/conda.sh /etc/profile.d/conda.sh && \
    echo ". /usr/local/anaconda3/etc/profile.d/conda.sh" >> ~/.bashrc

#Creates conda SRL environment
COPY origin /root/origin
RUN conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/ && \
    conda env create --file ~/origin/robotics-rl-srl/environment.yml && \
    echo "conda activate py35" >> ~/.bashrc && \
    pip install ipdb && \
    cp /usr/local/anaconda3/x86_64-conda_cos6-linux-gnu/sysroot/lib/libstdc++.so.6.0.25 /usr/lib/x86_64-linux-gnu/ && \
    rm -f /usr/lib/x86_64-linux-gnu/libstdc++.so.6 && \
    ln -s /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.25 /usr/lib/x86_64-linux-gnu/libstdc++.so.6

#Installs SSH
RUN apt-get update && apt-get install -y openssh-server net-tools sed supervisor && \
    mkdir -p /var/run/sshd && \
    mkdir -p /root/.ssh/ && \
    echo "root:12345678"|chpasswd && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
EXPOSE 22

#Installs MongoDB
ARG MONGO="mongodb-linux-x86_64-ubuntu1804-4.2.2"
RUN wget --quiet https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1804-4.2.2.tgz -O /usr/local/mongodb.tgz && \
    /usr/local/anaconda3/envs/py35/bin/pip install pymongo && \
    cd /usr/local && \
    tar zxvf /usr/local/mongodb.tgz && \
    rm /usr/local/mongodb.tgz && \
    mkdir -p /data/mongo/mongo_data && \
    chown -R 777 /data/mongo/mongo_data && \
    rm -rf /var/lib/apt/lists/*
COPY mongo.conf /usr/local/$MONGO/
ENV PATH=$PATH:/usr/local/$MONGO/bin
VOLUME ["/data"]
EXPOSE 27017

#Changes source for ROS downloading (Note:you can change the sources.list file to select a source suitable for your country)
COPY liblapack3_3.7.1-4ubuntu1_amd64.deb /root/
COPY libblas3_3.7.1-4ubuntu1_amd64.deb /root/
COPY libgfortran4_7.4.0-1ubuntu1_18.04.1_amd64.deb /root/
RUN cp /etc/apt/sources.list /etc/apt/sources.list.bak && rm /etc/apt/sources.list && \
    dpkg -i ~/libblas3_3.7.1-4ubuntu1_amd64.deb && dpkg -i ~/libgfortran4_7.4.0-1ubuntu1_18.04.1_amd64.deb && \
    dpkg -i ~/liblapack3_3.7.1-4ubuntu1_amd64.deb
COPY sources.list /etc/apt/

#Installs ROS
RUN apt-get update && apt-get install -y lsb-release dpkg && DEBIAN_FRONTEND=noninteractive apt install -y tzdata && \
        sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list' && \
        sudo apt-key adv --keyserver 'hkp://keyserver.ubuntu.com:80' --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654 && \
        apt-get update && apt-get install -y ros-melodic-desktop-full && rosdep init && rosdep update
RUN echo "source /opt/ros/melodic/setup.bash" >> ~/.bashrc && /bin/bash -c "source ~/.bashrc" && \ 
        apt install -y python-rosinstall python-rosinstall-generator python-wstool build-essential && \ 
        mkdir -p ~/SJTUSRL_ws/src

#Installs ROS2
RUN apt-get install -y locales && locale-gen en_US en_US.UTF-8 && \
    update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8 && \
    export LANG=en_US.UTF-8 && \
    apt-get install -y curl gnupg2 lsb-release && \
    curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add - && \
    sudo sh -c 'echo "deb [arch=amd64,arm64] http://packages.ros.org/ros2/ubuntu `lsb_release -cs` main" > /etc/apt/sources.list.d/ros2-latest.list' && \
    apt-get update && apt-get install -y ros-dashing-desktop ros-dashing-ros1-bridge

#Uses supervisord to open multi progress:sshd,mongodb
COPY supervisord.conf /etc/supervisor/
COPY supervisord_sshd.conf /etc/supervisor/
COPY supervisord_mongo.conf /etc/supervisor/
ENTRYPOINT ["/usr/bin/supervisord", "-nc", "/etc/supervisor/supervisord.conf"]

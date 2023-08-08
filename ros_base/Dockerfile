# Just Ubuntu 16.04 with a user called user and some basic tools
FROM ubuntu:16.04

RUN apt-get update
# Add sudo
RUN apt-get install apt-utils sudo -y

# Create user
RUN useradd --create-home --shell=/bin/bash user
RUN chown -R user /home/user/
# Add the user to sudoers
RUN chmod -R o-w /etc/sudoers.d/
RUN usermod -aG sudo user
# Give the user a password
RUN echo user:user | chpasswd

# Instal basic stuff
RUN apt-get install build-essential -y

# Nice tools to have
RUN apt-get install bash-completion nano net-tools less iputils-ping vim emacs -y
RUN apt-get install python-pip python-dev -y
# Download stuff
RUN apt-get install wget curl aria2 -y
# To enable ssh
RUN apt-get install openssh-server -y


WORKDIR /tmp
USER user

# Let's get some specs of the machine that is running this job
RUN cat /proc/cpuinfo; cat /proc/meminfo; df -h

# Note that we always use the same one
# This command will visit the latest release page and will look for the url of the artifact to download and download it
COPY ./gentoo_on_tmp.tar.lzma /tmp/gentoo_on_tmp.tar.lzma

# RUN aria2c -x 10 http://github.com`curl -s -L https://github.com/awesomebytes/gentoo_prefix_ci/releases/latest | grep "download/release" | cut -d '"' -f2`
# RUN aria2c -x 10 https://github.com/awesomebytes/gentoo_prefix_ci/releases/download/release%2F2019-01-07T18at09plus00at00/gentoo_on_tmp-amd64_2019-01-07T18at09plus00at00.tar.gz

RUN tar xvf gentoo_on_tmp.tar.lzma 
USER root
RUN rm gentoo_on_tmp.tar.lzma
USER user

# Make sure it works
RUN bash gentoo/executeonprefix emerge --version

# All commands to run in the Gentoo Prefix environment must
# be preprended by $PREFIXED
ENV PREFIXED=/tmp/gentoo/executeonprefix
ENV EPREFIX=/tmp/gentoo

# So we dont get complains
RUN echo "en_US.UTF-8 UTF-8" >> $EPREFIX/etc/locale.gen
RUN $PREFIXED locale-gen

# Let's make the compilations faster when possible
# Substitute the default -j2 with -j<NUM_CORES/2>
RUN rm -rf $EPREFIX/etc/portage/make.conf
RUN touch $EPREFIX/etc/portage/make.conf
RUN sed -i -e 's/j2/j'"$((`grep -c \^processor \/proc\/cpuinfo` / 2))"'/g' $EPREFIX/etc/portage/make.conf

# Add extra jobs if we have enough CPUs
RUN echo "EMERGE_DEFAULT_OPTS=\"--jobs $((`grep -c \^processor \/proc\/cpuinfo` / 2)) --load-average `grep -c \^processor \/proc\/cpuinfo`\"" >> $EPREFIX/etc/portage/make.conf
RUN cat $EPREFIX/etc/portage/make.conf

# dev-vcs/git-2.20.1 fails with (this command was already here, but executed after emerging git)
# ERROR: cannot verify www.kernel.org's certificate
# rosdep fails with
# ERROR: cannot verify pypi.org's certificate, issued by 'CN=DigiCert SHA2 Extended Validation Server CA,OU=www.digicert.com,O=DigiCert Inc,C=US':
#  Unable to locally verify the issuer's authority
# RUN $PREFIXED emerge dev-libs/openssl app-misc/ca-certificates net-misc/wget net-misc/curl
RUN $PREFIXED update-ca-certificates
RUN echo 'ACCEPT_LICENSE="*"' >> $EPREFIX/etc/portage/make.conf

# We got this on dev-libs/gobject-introspection, so we need to fix the softlink to gm4
# It's surprising it happens here now, as it was happening in the next phase beforehand
# 2020-03-09T13:29:38.1931278Z flex: fatal internal error, exec of /tmp/gentoo/usr/bin/gm4 failed
# 2020-03-09T13:29:38.1931639Z ninja: build stopped: subcommand failed.
# 2020-03-09T13:29:38.1953512Z  * ERROR: dev-libs/gobject-introspection-1.62.0::gentoo failed (compile phase):
RUN unlink $EPREFIX/usr/bin/gm4 \
    && ln -s $EPREFIX/usr/bin/m4 $EPREFIX/usr/bin/gm4

# Emerging git
RUN $PREFIXED emerge dev-vcs/git

# Setting EAPI to 6 as explained in http://wiki.ros.org/Installation/Gentoo
RUN mkdir -p $EPREFIX/etc/portage/profile/ && echo "8" > $EPREFIX/etc/portage/profile/eapi

# # Setting up ros-overlay https://github.com/ros/ros-overlay
RUN mkdir -p $EPREFIX/etc/portage/repos.conf 

# Set up Gentoo to sync via git (way faster)
RUN rm -rf $EPREFIX/usr/portage && echo "[DEFAULT]\n\
main-repo = gentoo\n\
\n\
[gentoo]\n\
location = /tmp/gentoo/usr/portage\n\
sync-type = git\n\
sync-uri = https://github.com/gentoo-mirror/gentoo \n\
auto-sync = yes\n\
masters = gentoo" > $EPREFIX/etc/portage/repos.conf/gentoo.conf

# Sync everything
RUN $PREFIXED emaint sync -a

# # bug 674830
# RUN echo ">=sys-apps/dbus-1.12.10" >> $EPREFIX/etc/portage/package.mask
RUN $PREFIXED emerge sys-apps/dbus


# Due to circular dependencies:
#  * Error: circular dependencies:
# net-wireless/bluez-5.54:0/3::gentoo, ebuild scheduled for merge) depends on
#  (dev-libs/glib-2.62.5:2/2::gentoo, ebuild scheduled for merge) (buildtime)
#   (dev-util/meson-0.53.1:0/0::gentoo, ebuild scheduled for merge) (buildtime)
#    (dev-python/setuptools-44.0.0:0/0::gentoo, ebuild scheduled for merge) (buildtime)
#     (dev-lang/python-2.7.17-r1:2.7/2.7::gentoo, ebuild scheduled for merge) (buildtime)
#      (net-wireless/bluez-5.54:0/3::gentoo, ebuild scheduled for merge) (buildtime)
# We build without bluetooth first, then with... (note that the bluetooth support is needed
# because some ROS package use bluetooth from Python)
RUN $PREFIXED emerge python:3.10
# Build python with sqlite and bluetooth support
# I find very surprising sqlite is not mandatory
# Also enable PGO and LTO optimizations
# RUN echo "dev-lang/python sqlite bluetooth readline" >> $EPREFIX/etc/portage/package.use
# To build succesfully the bluetooth support
RUN echo "net-wireless/bluez -udev" >> $EPREFIX/etc/portage/package.use &&\
    echo ">=dev-libs/libxml2-2.9.9-r3 python" >> $EPREFIX/etc/portage/package.use
# RUN EXTRA_ECONF="--enable-optimizations --with-lto" $PREFIXED emerge python:3.10[sqlite,bluetooth,readline]

# Latest boost has an issue https://github.com/ros/ros-overlay/issues/986
# So we mask it for now so we can build roscpp

# Copy lib from ros-overlay then create manifest 
ADD --chown=user:user dev-util/boost-build $EPREFIX/usr/local/portage/dev-util/boost-build
RUN $PREFIXED ebuild $EPREFIX/usr/local/portage/dev-util/boost-build/boost-build-1.78.0-r1.ebuild manifest && \
    $PREFIXED emerge =dev-util/boost-build-1.78.0-r1

ADD --chown=user:user dev-libs/boost $EPREFIX/usr/local/portage/dev-libs/boost
RUN $PREFIXED ebuild $EPREFIX/usr/local/portage/dev-libs/boost/boost-1.78.0-r3.ebuild manifest && \
    $PREFIXED emerge =dev-libs/boost-1.78.0-r3

# Emerge llvm already, as it takes 2h30 already, so we can
# debug the next step (where we really emerge ros_base) easier and faster
# RUN $PREFIXED emerge sys-devel/llvm

#Install ROS2 python dependencies
RUN $PREFIXED emerge -n app-eselect/eselect-python
RUN $PREFIXED eselect python set python3.10
RUN $PREFIXED eselect python cleanup && $PREFIXED  eselect python list
RUN echo "*/* PYTHON_TARGETS: python3_10\n\
*/* PYTHON_SINGLE_TARGET: -* python3_10\n\
" >> $EPREFIX/etc/portage/package.use

# mask python 3.11
RUN echo "dev-lang/python:3.11" >> $EPREFIX/etc/portage/package.mask

RUN $PREFIXED emerge --changed-use --deep --exclude dev-libs/libical @world
RUN ${PREFIXED} emerge --depclean dev-lang/python && ${PREFIXED} eselect python cleanup && ${PREFIXED}  eselect python list
RUN ${PREFIXED} emerge dev-python/pip

RUN $PREFIXED USE='-rsync-verify' emerge -qvO portage && $PREFIXED emerge -qv portage

RUN $PREFIXED PYTHON_TARGETS="python3_10" emerge --pretend -v1 sys-apps/portage

RUN $PREFIXED emerge -1 portage

RUN ${PREFIXED} python3.10 -m ensurepip --upgrade 
RUN ${PREFIXED} python3 -m pip install --user  opencv-python colcon-common-extensions coverage flake8 flake8-blind-except flake8-builtins flake8-class-newline flake8-comprehensions flake8-deprecated flake8-docstrings flake8-import-order flake8-quotes mock mypy pep8 pydocstyle pytest pytest-mock vcstool empy lark

#Install asio (Required by ROS2)
RUN cd ~/ &&\
 wget https://sourceforge.net/projects/asio/files/asio/1.28.0%20%28Stable%29/asio-1.28.0.tar.gz &&\
 tar -xzf asio-1.28.0.tar.gz &&\
 cd asio-1.28.0/ &&\
 autoreconf -i &&\
 ./configure &&\
 make &&\
 make prefix=/tmp/gentoo/usr libdir=/tmp/gentoo/lib install 

#Download ROS2
RUN mkdir -p ~/ros2_humble/src &&\
    cd ~/ros2_humble &&\
    vcs import --input https://raw.githubusercontent.com/ros2/ros2/humble/ros2.repos src &&\
    #Remove unecessary packages (Unusable display and DDS implementations that do not work on 32 bits OS)
    rm -rf src/eclipse-cyclonedds src/eclipse-iceoryx src/ros-visualization src/ros2/rviz src/ros/ros_tutorials/turtlesim &&\
    #Build ROS2
    colcon build --symlink-install

#Download naoqi_driver
RUN . ~/ros2_humble/install/local_setup.bash &&\
    mkdir -p ~/catkin_ros2/src &&\
    cd ~/catkin_ros2/src/ &&\
    git clone https://github.com/RoboBreizh-RoboCup-Home/naoqi_driver2.git  &&\
    git clone --branch humble https://github.com/ros-perception/vision_opencv &&\  
    git clone --branch debian/galactic/naoqi_libqi https://github.com/ros-naoqi/libqi-release &&\
    git clone --branch debian/galactic/naoqi_libqicore https://github.com/ros-naoqi/libqicore-release &&\             
    git clone https://github.com/ros-naoqi/naoqi_bridge_msgs2 &&\
    git clone https://github.com/ros/diagnostics


#Compile naoqi_driver
RUN cd ~/catkin_ros2 &&\
    . ~/ros2_humble/install/local_setup.bash &&\
    colcon build --symlink-install


ENTRYPOINT ["/bin/bash"]

mkdir /opt/downloads
cd /opt/downloads
yum install git -y
git clone https://github.com/vozlt/nginx-module-vts.git


git clone https://github.com/vozlt/nginx-module-sts.git

git clone https://github.com/vozlt/nginx-module-stream-sts.git

yum -y install gcc gcc-c++ make zlib-devel pcre-devel openssl-devel git wget geoip-devel epel-release

yum autoremove nginx
wget http://nginx.org/download/nginx-1.13.0.tar.gz
tar -zxf nginx-1.13.0.tar.gz
cd nginx-1.13.0

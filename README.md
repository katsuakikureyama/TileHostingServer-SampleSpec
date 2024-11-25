# Features
## Tools 
- Select a prefecture in Japan to download map tiles from multiple sources
- Compression of map tile images (converted to jpg)  

## Server  

- Map server for self-hosting
- Map server for proxy from source 
- Server and browser cache settings 

## Test UI 

A sample UI using OpenLayers.js is available by setting the environment variable TEST_UI to true.

This displays a map based on "http://localhost:8080/config/tiles-config.json".
You can refer to the source for UI creation.

URL: http://localhost:8080/test/ol 
Sources /src/server/app/uitest/ol

# Build 
We created an application on docker. 
You can download image from docker hub repository .

- "/tileserver-tools" 
- "/tileserver" 

But also, You can build the image from source.

## tileserver-tools 
When you execute this image, The MapTile will be installed to the directory.

## tileserver
When you execute this image , The MapTile will be published from instance. 
See

## Use Docker Compose
I recommend to build using DockerCompose.
You can use those files when you use docker compose.

### for example 
```
###
#
#  Build from docker hub repository
#    docker-compose-init.yml  
#    docker-compose-release-server.yml 
#
#  Build from soruce
#    docker-compose-src-init.yml  
#    docker-compose-src-release-server.yml 
#
###

## download the map tiles 

docker compose -f  docker-compose-init.yml up  
docker compose -f  docker-compose-init.yml down 

## start up 

docker compose -f  docker-compose-release-server up

## If you finished to pre-download the maptiles , the image is no longer necessary.
## You can remove the image and resouces using this command for saving the host storage.

## Remove tileserver-tools image 
docker image rm /tileserver-tools -f 

## Remove resouces 
find /path/to/your/docker-compose-directory -maxdepth 1 ! -name 'docker-compose-release-server.yml' ! -name '.' -type f -exec rm -f {} +

```

### res-volume

The map tiles retrieved by tileserver-tools are shared to the tileserver through common storage.
However, mounting the host file system to the docker container has a large processing overhead, so the DockerCompose build we have prepared utilizes volume mounts.
This volume is managed by Docker.

## Custom configuration

### config/tiles-config.json
The configuration of tile server-tools depends on tiles-config.json which is included the tile settings, descriptions.
This file is also published on tile server for front-end. 
http://localhost:8080/res/tiles-config.json

```
{
    "extractings" : [

         {
            //  Title for using by front-end.
            "title":"OpenStreetMapTile", 
            //  Description for using by front-end.
            "description": "OpenStreetMapのタイルです。",  
            //  Copyright notice for using by front-end.            
            "source":"© <a href='http://osm.org/copyright'>OpenStreetMap</a> contributors, <a href='http://creativecommons.org/licenses/by-sa/2.0/'>CC-BY-SA</a>'", 
            // Tile Sources 
            "baseurl": "http://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
            // URL path  to save tiles 
            "destdir": "/res/tiles/osm",
            // Multiple prefectures can be specified, separated by commas. 
            "prefectures": "東京都,大阪府",
            // Tile zoom range
            "zoom": "0:14",
            // If you don't convert the tile-image to jpg, it is not necessary.  
            "jpeg": { "quality": 20 }
        },
        {
            "title" :"年度別空中写真 1979年～1983年",  
            "description": "国土地理院からのタイルです。", 
            "baseurl": "https://cyberjapandata.gsi.go.jp/xyz/gazo2/{z}/{x}/{y}.jpg",  
            "source":"出典:国土地理院 https://maps.gsi.go.jp/development/ichiran.html", 
            "destdir":  "/res/tiles/1979-1983",  
            "prefectures": "東京都",  
            "zoom": "10:17",
            "jpeg": { "quality": 30 }.  
        },
        {
            "title" : "国土地理院 土地条件図 初期整備版",
            "description": "この地理院タイルは基本測量成果（名称：地理院タイル（土地条件図））です。",
            "source":"出典:国土地理院 https://maps.gsi.go.jp/development/ichiran.html",
            "baseurl": "https://cyberjapandata.gsi.go.jp/xyz/lcm25k/{z}/{x}/{y}.png",
            "destdir": "/res/tiles/lcm25k",
            "prefectures": "東京都",
            "zoom": "10:16",
        }
    ]
}
```

### docker-compose-release-server.yml

The configuration of tile server depends on the environment-variable which is included memory-limit ,cache-setting.
The environment-variable can be set by docker-compose.yml .

```
version: '3'
services:

  server:
    image: /tileserver:latest
    ports:
      - "8080:3000"
    tty: true
    volumes:
      - res-volume:/res
    environment: 
       - NODE_MAX_OLD_SPACE_SIZE=256  # Max memory size for the application. 
      
      - USE_PUBLIC_DIRECTORY=false # If you want to access static files in "/app" from the client, you can do so from "http://localhost:8080/".
      - TEST_UI=false # You can access the test UI  http://localhost:8080/test/ol/ 
      
      - SERVER_CACHE_MAX_AGE=1 day         # The server-cache setting depends on "https://github.com/kwhitley/apicache". 
      - CLIENT_CACHE_MAX_AGE=31536000  # You can set the Cache-Control for the ResponseHttp header. Unit is seconds.
      
      - USE_TILE_PROXY=false # Tiles are retrieved from the resource server when accessed by a client.
      - STORE_TILE_BY_TILE_PROXY=false # Tiles retrieved from the resource server are stored in storage and utilized.
      
      - STORED_TILE_EXPIRE_SEC=3600  # After the expiration time, the tile is retrieved again.
      - STORED_TILE_MAX_BYTE=500000000 # Maximum total stored tile size .
    
    command: npm start 

volumes:
  res-volume:
```
 
## Create an environment for Ubuntu

### Install docker 
```
sudo apt update 
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y docker-ce
sudo systemctl start docker
sudo systemctl enable docker

sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version

```

## Register the as an auto-start app in systemctl

Make a service file.

```
#/etc/systemd/system/docker-compose-tileserver.service
#Please replace "/path/to/your/" to your directory.

[Unit]
Description=Docker Compose Application Service
After=network.target docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/linux-startup.sh --dir=/path/to/your/docker-compose-directory --config=release-server
ExecStop=/usr/local/bin/docker-compose -f /path/to/your/docker-compose-directory/docker-compose-server-release.yml down
WorkingDirectory=/path/to/your/docker-compose-directory

[Install]
WantedBy=multi-user.target

```

Here is startup command. 

```
sudo systemctl daemon-reload
sudo systemctl enable docker-compose-app
sudo systemctl start docker-compose-app

```





# TileHostingServer

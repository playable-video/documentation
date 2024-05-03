# Add video API

## 0. Acquire access token

API definition:

```
POST https://api.playable.video/1/session
Body: { "email": String, "password": String }
Response: { "access_token": String, properties: [String] }
```

Example request:

```
username=""
password=""
url="https://api.playable.video/1/session?lang=en"
curl -X POST -H "Content-Type: application/json" -d '{"email":"$username","password":"$password"}' "$url" > session.json
account_id=$(cat session.json | jq '.properties | .[0]' -r)
access_token=$(cat session.json | jq .access_token -r)
```

## 1. Create an Edit

API definition:

```
POST https://api.playable.video/1/edit?lang=en
Body: {
  "stem":"autoplay",
  "params":{},
  "property_id": String, // property id on the console
  "file": String, // file name
  "content_type":"video/quicktime",
  "source":{
    "duration": Number, // duration in seconds
    "best": {
      "width": Number, // desired width in pixels
      "height": Number, // desired height in pixels
      "crop":"$width:$height:0:0", 
      "filesize": Number // desired size in bytes
    }
  },
  "access_token":"$accessToken" //Access token from step 0
}

Response: {
    "edit_id": Number,
    "url_upload": String, //Signed URL to upload to S3
    "edit": {
        "states": {
            "autoplay": {
                "status": String, // Current status of the edit process
                "details": String // Details on the status
            }
        },
    }
}
```

Example request:

```
property="samples.playable.video"
read -r -d '' requestBody << EOM
{
  "stem":"autoplay",
  "params":{},
  "property_id":"$property",
  "file":"302025161-b89a3e71-18cf-4909-86aa-911fe355e68d.mov",
  "content_type":"video/quicktime",
  "source":{
    "duration":8.683333,
    "best":{"width":3024,"height":1646,"crop":"3024:1646:0:0","filesize":1918907}
  },
  "access_token":"$accessToken"
}
EOM
url="https://api.playable.video/1/edit?lang=en"
curl -X POST -H "Content-Type: application/json" -d "$requestBody" "$url" > edit.json
```

## 2. Create the video

```
POST https://api.playable.video/1/video?lang=en
Body: {
  "width": Number, // width in pixels of the source video
  "height": Number, // height in pixels of the source video
  "loop": 0, // Number of times the video should loop
  "auto_height": true,
  "title": String, // Title for the video
  "edit_id": Number, // Edit id from step 1
  "access_token": String // Access token from step 0
}
Response: {
  "video": {
    "updated": Timestamp,
    "snippets": {
      "inline": String // Snippet to attach to ESPs
    },
  },
  "video_id": String // Video id
}
```

Example request:

```
curl --verbose -H "content-type: application/json" \
  --data '{"title":"Test62","video_id":"blizzard.playable.video/v:5635971563388928","stream_manifest_url":"http://content.uplynk.com/channel/ext/d09b16c953aa40c98dd8c513526aca5a/a118914630.m3u8?ad.kv=_fw_ae,d8b58f7bfce28eefcc1cdd5b95c3b663&exp=1464539209&ptid=ESPN3_Events_VDMS&rn=961076378&tc=1&oid=d09b16c953aa40c98dd8c513526aca5a&linearv=4&ad=e3ads&ad.csid=watchespn:desktop:e3ads&ad.caid=a118914630&eid=a118914630&euid=ESPN3_VDMS&ct=c&sig=c29805f5fd81fa6d924876ecd695874dc36504458a593986c8250d85c7a8fd56","access_token":"5762273213677568-6d445961da87c36166061f9dda5fef14cd7ca91556d0314a1c34ad5b3f85a1d2"}' \
  https://vidiense-dev.appspot.com/1/video
property="samples.playable.video"
read -r -d '' requestBody << EOM
{
  "width":600,
  "height":338,
  "loop":0,
  "auto_height":true,
  "title":"302025161-b89a3e71-18cf-4909-86aa-911fe355e68d",
  "edit_id":5665350314885120,
  "access_token":"$accessToken"
}
EOM
url="https://api.playable.video/1/video?lang=en"
curl -X POST -H "Content-Type: application/json" -d "$requestBody" "$url" > video.json
```

## 3. Upload video

Use the signed S3 url from step 1 to upload the video to S3 using any S3 client.

## 4. Check status

```
GET https://api.playable.video/1/edit/$editId?access_token=$accessToken&lang=en&src=firebase

Response: {
    "edit": {
        "states": {
            "autoplay": {
                "status": String, // Current status of the edit process
                "details": String // Details on the status
            },
        },
    }
}
```

The video will be ready to send when `$.edit.states.autoplay.status` becomes `ready`. Other possible values are 
`uploading`, `compiling`, and `transcoding`.

## 5. Get the snippet

Using the video ID from step 2:

```
GET https://api.playable.video/1/video/$videoId?access_token=$accessToken&lang=en&src=firebase

Response: {
  "video": {
    "snippet_html": String, //video html
  },
  "video_id": "samples.playable.video/v:5665350314885120",
}
```

The html to send to the ESP is in `$.video.snippet_html`.
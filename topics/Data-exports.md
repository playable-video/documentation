# Data Exports

Playable supports exporting report data through a REST API. There are 15 types of report based the partition key of the data. Regardless of the key of the data, the rest of the schema contains the same columns:

- `open`: The number of times the video started playing.
- `reopen`: The number of times a customer played a video they already watched.
- `billable_plays`: The number of plays that Playble will actually charge the account on. This number is the number of `open` minus the number of Prefetch requests.
- `click`: The number of times a customer clicked in a video.
- `reclick`: The number of times a customer clicked in a video they already clicked on.
- `native`: The number of times a customer started playing a video on a native app (that is, an email client app in a touch device).
- `renative`: The number of times a customer started playing a video on a native app that they have already played.
- `disconnect`: The number of times a customer closed a video half way through.
- `redisconnect`: The number of times a customer closed a video half way through that they have already played.

These are the possible keys to request:

- `app`: The key is the name of the email client used to watch the video.
- `os`: The key is the operative system of the customer.
- `osversion`: The key is the version of the operative system of the customer.
- `mime`: The key is the format of the video delivered to the customer.
- `network`: The key is the type of network used to fetch the video (wifi, data, etc).
- `density`: The key is the resolution of the video delivered to the customer.
- `date`: The key is the date of each play.
- `continent`: The key is the continent of the customer at the time of playing.
- `country`: The key is the country of the customer at the time of playing.
- `region`: The key is the region within the countr at the time of playing.
- `city`: The key is the city of the customer at the time of playing.
- `hod`: The key is the hour of the day at which the video was played.
- `dow`: The key is the day of the week in which the video pas played.
- `language`: The key is the language tag of the video played (contact support@playable.video to know how to use this feature).
- `variation`: The key is the specific variation of an A/B tested delivered to the customer (contact support@playable.video to know how to use this feature).

## How to export reports as CSVs

Playable uses access tokens to authenticate all API requests. In order to get your reports, you have to first create an
access token to validate your interactions.

You will be using three endpoints. All post requests use JSON bodies:

```
POST https://api.playable.video/1/session
Body: { "email": String, "password": String }
Response: { "access_token": String, properties: [String] }

GET /1/daily-report/$account?report_id=$key&access_token=$access_token
Response: CSV file with activity of $account partitioned by $key

GET /1/daily-report/$account/$video_id?report_id=$key&access_token=$access_token
Response: CSV file with activity of video $video_id of $account partitioned by $key
```

You can retrieve all the video ids of an account using this API:

```
GET /1/property/$account?access_token=$access_token
Response: { "videos": [{"key": String}] }
```

This is an example of how to use the API:

```bash
username=""
password=""
url="https://api.playable.video/1/session?lang=en"
curl -X POST -H "Content-Type: application/json" -d '{"email":"$username","password":"$password"}' "$url" > session.json
account_id=$(cat session.json | jq '.properties | .[0]' -r)
access_token=$(cat session.json | jq .access_token -r)
curl "https://api.playable.video/1/daily-report/$account_id?report_id=date&access_token=$access_token" > report_by_date.csv
for video_key in $(curl "https://api.playable.video/1/property/$account_id?access_token=$access_token" | jq '.videos | .[] | .key' -r);
do
  # Note that $video_key already contains the property path, i.e. it is video_key=$account_id+"/"+$video_id
  mkdir -p $video_key
  curl "https://api.playable.video/1/daily-report/$video_key?report_id=date&access_token=$access_token" > $video_key/report_by_date.csv
done
```
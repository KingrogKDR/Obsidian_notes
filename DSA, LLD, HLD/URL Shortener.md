## Functional Requirements

- one unique short url for one long url 
- the short url only expires after the user prompts to do so.
- the user can have custom aliases in their short url
- the user should be able to view the analytics on that short url, for e.g the no. of clicks on it, the demographics and geographics of the user clicking on it, etc.
## Non-functional Requirements

- the redirection should be fast.
- the short url should have 100% availability with respect to the long url, i.e the short url only shouldn't work when the long url is down.
- reads will always be greater than writes, because the no. of short urls generated is much less than the no. of short urls clicked.
## Back of the envelope estimation

- 100M urls/month, read/write ratio of 100:1

- 1 sec -> 100M/(30\*24\*3600) urls ~  38 new urls/s  -> 38 writes/s and 3800 reads/s

- 500 bytes/record -> 38\*500 bytes/s -> 19000 bytes/s -> 68.4 mb/hr ~ 1.6 gb/day ~ 48gb/month ~ 576 gb/year ~ 2.8 tb in 5 years
## Api Design

- POST /api/v1/create -> creates a short url for a given long url

Request body: 
```json
{
	"long_url" : "url_content"
}
```

Response: 201 
```json
{
	"short_url": "short_url_content"
}
```

- DELETE /api/v1/short_url -> deletes a short url

Request body: {}

Response: 200
```json
{
	"message": "short_url %s successfully deleted"
}
```

- GET /api/v1/redirect -> gets the long url from the short url

Request Body:
```json
{
	"short_url": "short_url_content"
}
```

Response: 200
```json
{
	"long_url" : "url_content"
}
```

Schema:

URL Table
```sql
Short_url VARCHAR(256) UNIQUE NOT NULL
Long_url VARCHAR(256) NOT NULL
```



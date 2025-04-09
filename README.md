# First try using wikimedia API

## Ask Gemini

Can you give me an example curl line for getting a summary of a given
wikipedia page (say Scorpion) using Wikimedia REST API ?

## Refine and use example

```bash
UA='User-Agent: thy/1.0 mailto:t.delamare@laposte.net'
url=https://en.wikipedia.org
page=Scorpion

curl -sH "$UA" $url/api/rest_v1/page/summary/$page | jq
```

# Second try

## Ask Gemini

Can I get a wikipedia list page (e.g. `List_of_French_philosophers`) as json ?

## Refine and use example

```bash
list=List_of_French_philosophers
```

### Get page

```bash
data=(action parse page $list prop text format json formatversion 2)
jq='$ARGS.positional | _nwise(2) | "--data-urlencode \(first)=\(last)"'
args=$(jq "$jq" -nr --args ${data[@]})

curl -sH "$UA" -G $url/w/api.php $args > tmp/$list-1.js
```

### Extract list

```bash
jq='inputs | select(test("<li><a href=\"/wiki/"))'
jq+='| capture("href=..(?<href>[^\"]+).*title=.(?<title>[^\"]+)")'

< tmp/$list-1.js jq -r .parse.text | jq -Rn "$jq" > tmp/$list-2.js
```

### Generate requests from list

```bash
jq='"curl -sLH \"\($ua)\" \($url)/api/rest_v1/page/summary/\($q)\((.href / "/")[-1])\($q)"'

mkdir -p tmp/$list/jpg
< tmp/$list-2.js jq "$jq" -r --arg ua "$UA" --arg url $url --arg q "'" > tmp/$list-3.sh
```

### Get all summary

```bash
< tmp/$list-3.sh bash | jq > tmp/$list-4.js
```

### Generate request for all thumbnails from summaries

```bash
jq='select(has("thumbnail")) | "curl -s \(.thumbnail.source) -o \"\($list)/\(.title | sub(" "; "_"; "g")).jpg\""'
jq='select(has("thumbnail")) | "curl -s \(.thumbnail.source) -o \"\($list)/jpg/\(.titles.canonical).jpg\""'

< tmp/$list-4.js jq "$jq" --arg list tmp/$list -r > tmp/$list-5.sh
```

### Get all Thumbnails

```bash
< tmp/$list-5.sh bash
```

### Generate a MD file

```bash
jq='"# \(.title)", "", .extract, if has("thumbnail") then "", "![\(.title)](jpg/\(.titles.canonical).jpg)" else empty end, ""'

< tmp/$list-4.js jq "$jq" -r > tmp/$list/README.md
```

### Keep MD and JPG

```bash
mkdir -p out
mv tmp/$list out
```

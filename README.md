# fis_in_depth
In depth explanation of the input system

FIS (aka Fancy Input System) is using this CLI app helper called 'meow' (https://www.npmjs.com/package/meow).

We’re mainly using FIS when we want to include specific streams or dates as input in our scrapers. And it makes choosing specific streams very easy.

Usage:
So we're using it by calling:
node scriptName.js [options] [streams_to_include]

For options, there are few choices

We can use --all ( '--' before all are very important since that’s how FIS recognizes the specific command) which captures all available streams. 

For dates, we're using --from followed by --to to set a date range.
We can use common dates like 'yesterday', 'tomorrow' or 'today'. Or even insert the specific date in the ISO date format.
For testing I'm usually using something like this: 
node scriptName.js --all --from yesterday --to today
It captures all available streams from yesterday to tomorrow.

Here are few short examples with common combinations:
```
node script.js stream_a stream_b
^- Capture stream_a and stream_b, default dates
node script.js --all --exclude stream_a --exclude stream_b
^- Capture all available streams except stream_a and stream_b
node script.js --all --tz America/Chicago --from yesterday --to tomorrow
^- Capture all available streams from yesterday start of day to tomorrow start of day in Chicago
node script.js --all --from "2019-12-15T10:00:00.000Z" --to today
^- Capture all available streams from 15th Jan 2019 10:00 UTC to start of today default time zone
```
So here's a short cheatsheet with all options:
```
  --all                        Includes all streams
  --exclude=...                If "--all" passed, excludes specified stream
  --from=...                   Overrides "from" default date (default "from" date decided by the scraper)
  --to=...                     Overrides "to" default date (default "from" date decided by the scraper)
  --tz=...                     Overrides default timezone (default timezone decided by the scraper)
  --info                       Show available streams and defaults of the scraper
```

How do we call it from scraper itself?
```javascript 
const { fis } = require('@incom/data-entry-js-sdk'); // to use it from our SDK
const input = fis(availableStreams, {
    defaultFrom: 'yesterday', defaultTo: 'tomorrow', defaultTz: 'UTC', onlyOne: false,
  });
```
availableStreams must be an array and onlyOne is a flag, if it is set to true then it limits the output of streams to 1( so --all flag can't be used together with onlyOne)

We can access user's input with:
```javascript
const streams = input.streams; // to get an array of all chosen streams
const dateRange = input.dateRange; // to get chosen date range in Luxon's DateTime format
const from = dateRange.from;
const to = dateRange.to;
```
And the most important, an example of scraper which is using FIS:
```javascript 
const { fis } = require('@incom/data-entry-js-sdk');

const streams = {
  streamA: [0, 'streamid_1'],
  streamB: [1, 'streamid_2'],
  streamC: [2, 'streamid_3'],
  streamD: [3, 'streamid_4'],
};

(async () => {
  const input = fis(Object.keys(streams), {
    defaultFrom: 'today',
    defaultTo: 'tomorrow',
    defaultTz: 'UTC',
    onlyOne: false,
  });
  const from = input.dateRange.from.toFormat("yyyy-MM-dd'T'HHmmss");
  const to = input.dateRange.to.toFormat("yyyy-MM-dd'T'HHmmss");
  
  const sourceData = await requestATCData(parseUrl(from, to, 'CZ'));

  const insertions = 
        Object.entries(streams)
          .filter(([k]) => input.streams.includes(k)) // here we go through all chosen streams
          .map(async ([k, i]) => {
            const idx = i[0];
            const streamId = i[1];
            
            const dWithNulls = sourceData.map((x) => ({ date: parseDate(x.DF, x.TF), value: getValue(x, idx) }));

            const d = dWithNulls.filter((x) => x.value !== null);
            if (d.length === 0) {
              logger.info('Empty data set for stream %s, streamId %s', k, streamId);
              return Promise.resolve();
            }

            return constructAndSubmitRecord(streamId, 'ts1', d);
          })
          .map((i) => i.catch((e) => logger.error(e.message)));

  await Promise.all(insertions);
})();
```


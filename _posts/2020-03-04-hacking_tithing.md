# Hacking tithing

I was asked to manage finances in my local congregation for a while. Our church has a centralized system which manages finances. Naturally, I wanted more functionality than their interface provided, such as retrieval of entire batches of data at once, so I could do my own data munging and not rely on their watered-down routines.

The batches I had permission to download was all visible from the church financials page, after maximizing the date range in the transaction summary screen. So I would just capture all the text from that page and dump it into a text file ("batches.txt"). The url for the itemized data was formatted as follows:

`https://beta.lds.org/finance/donations/slips?donationBatchId=[batch number]&internalAccountId=[numeric user id]`

I found my numeric user id just by snooping the urls used when accessing itemized data. The batch ids had the following pattern:

`batch_pattern = re.compile('[0-9]{8}')`

So it was easy enough to grab the batches from the transaction summary screen and output them to an intermediate file, with code like this

```
  batches = []
  for line in open(fromFile, 'r'): # just text copied from donations page with custom dates to what we want
    match = batch_pattern.search(line)
    if match:
      batches.append(match.group(0))
```

I didn't want to deal with secure logins and session information in my scripts, so I used the "Chrono Download Manager" on Chrome as follows. I took the batch names and transformed them into the urls I wanted and put those in a file, one per line. I copied the contents of the file to the clipboard. The download manager can grab all the urls on the clipboard and download them all at once. I used `*query*` as the naming mask so batch number info was saved. After saving all the files, I moved and renamed them to just the batch number, as batch numbers are unique:

```
def fixBatchNames(fromFolder='batches_precleaned',toFolder='batches'):
  p = Path('.') / fromFolder
  t = Path('.') / toFolder
  for path in p.iterdir():
    match = batch_pattern.search(str(path))
    if match:
      shutil.move(str(path),str(t / match.group(0)))
```

Batch files were stored in a simple JSON format. I could extract all the relevant data into a csv format with the following:

```
def compileData(fromFolder='batches',toFile='totals.csv'):
  p = Path('.') / fromFolder
  data = []
  for path in p.iterdir():
    match = batch_pattern.search(str(path))
    if match:
      with open(str(path),'r') as f:
        data += json.load(f)
  summaries = defaultdict(lambda:defaultdict(lambda:defaultdict(lambda:0))) # name, year, values
  for d in data:
    summaries[d['name']][d['donationDate'][:4]]['total'] += d['amount']
    for line in d['lines']:
      if 'Tithing' in line['subcategory']['category']:
        summaries[d['name']][d['donationDate'][:4]]['tithing'] += line['amount']
      elif 'Fast Offering' in line['subcategory']['category']:
        summaries[d['name']][d['donationDate'][:4]]['fast offering'] += line['amount']
```

I could also extract donation information for a single patron with the following

```
def dataFor(name,fromFolder='batches'):
  name = name.lower()
  p = Path('.') / fromFolder
  data = []
  for path in p.iterdir():
    match = batch_pattern.search(str(path))
    if match:
      with open(str(path),'r') as f:
        data += json.load(f)
  return sorted([[d['donationDate'],*[line['amount'] for line in d['lines'] if 'Tithing' in line['subcategory']['category']],d['amount']] for d in data if name in d['name'].lower()])

  sums = []
  for name in summaries:
    for year,items in summaries[name].items():
      sums.append({'name':name,'year':year,'tithing':items['tithing'],
                   'fast offering':items['fast offering'],'total':items['total']})
  sums.sort(key=lambda x:x['name']+x['year'])
  with open(toFile,'w',newline='') as f:
    writer = csv.DictWriter(f,fieldnames=['name','year','tithing','fast offering','total'],restval='0')
    writer.writeheader()
    writer.writerows(sums)
```

All in all, pretty simple. Also pretty useful.

+++
title = 'The Pedestrian Yet Subtle Art of Csv Ingestion'
date = 2024-09-23T20:35:41+02:00
draft = false
+++
# The Pedestrian Yet Subtle Art of CSV Ingestion

**TL;DR:** CSVs (or worse yet, excel files) offer a poor choice of data format for an API between two systems. Despite that, there is sometimes no escaping them. In my opinion, when this situation arises there are some key principles to be followed to avoid the transfer of these files being a constant source of busy work for the receiver. In practice, I rarely see these principles obeyed by and believe there is a gap in the available tooling for this specific use case.

## Background
For personal reasons (moving country and a baby arriving) I have changed employer twice in short succession recently meaning there was a window where I had worked for three different employers over a window of about a year. Moving from one company to another and then another in short succession allows you to see certain recurring issues and problems sometimes. One thing I noticed is that in all three of these employers there was at least one process which ingested CSV or excel files provided by a third party via FTP or email.

Now I know, this is 2024 and ChatGPT can read my mind already so this seems like a very unfashionable topic for a blog article. But what I noticed from seeing this situation multiple times is that it is a problem which is generally solved badly. Whilst, like any developer, I'm excited by new technology, the real benefits of new technology will never reach society as a whole when most companies can't get basic stuff right.

## CSV as an API
So, whats the problem? The fundamental problem with this kind of set up is that CSV is a bad choice of format to act as an API between two systems. Because remember, if it's the output of one system and in the input to another system, that's an API - it should be treated like one. In this scenario CSV is a bad format because, lacking types and schema, it offers no way to define a formal set of expectations on the data being transferred nor a method with which to verify the data arriving is as expected. 

In lieu of this, informal, poorly defined expectations of the data are often used with work often scoped out using example files which may not capture edge cases and often do not accurately reflect the final data format after go-live. Furthermore, in the scenario where CSV is the only available format (usually due to a lack of knowledge or resourcing on the data producer side), the technologies used on the producer side (usually a off the shelf reporting tools, excel, sql queries or a combination of these) can often be a source of ongoing drift in the data format.

Because of the lack of a formal format, and the ability of the format to drift, the import and export side of the transfer process frequently do not align meaning that the import process errors frequently. Due to the the fact the transfer protocol is usually FTP or email, it makes it difficult for the receiver of the data to push that error left to the data producer. 

Without a formal validation step, bad data may just cause unhandled exceptions in the import process or downstream processes that use the data; but worse, they may silently pass through all processes but invoke unintended behaviour or display invalid data to end users. These downstream issues can typically be very hard to debug. 

In it's worst incarnation this can lead to an "automatic" import of data that requires so much ongoing maintenance it actually uses more dev time than manually importing the data would have done.

## The Best Solution: Don't do it
The first, most sure-fire way to prevent these problems is to not set up this kind of data import process in the first place. If it is asked of you, pull all the organizational levers you can to do it another way. 

The most obvious alternative is a web api set up on your side for the data producer to post the data to: this means data can come in as json (with actual types!), the structure of the data can be validated in real time and rejected if it doesn't meet the cut. This puts pressure back onto the data producer to provide consistent, good quality data. This might sound a like a lot of work for a single import, but these days if you have a contract with a cloud provider it's really not that difficult to set up an API gateway and Lambda function (or equivalent) to process such requests.

If this isn't an option, see if you can at least get the data producer to provide the data in a better format, Avro or Parquet preferably, but json with an agreed [json schema](https://json-schema.org/learn/getting-started-step-by-step) can also do the job. The schema can then be agreed with the producers in the scoping stage of the project and provide an unambiguous description of the data than can be used by both the data producers and the consumers when serializing/deserializing the data.

However, unfortunately, the same reason these processes pop up in the first place is the same reason it's hard to design an alternative. Usually, data is being transferred as a CSV on an email because the data producers basically don't have the expertise and/or resource to do anything better. If you ask them to make a post request or provide a Parquet file you may quickly hit a brick wall.

For the situation of manual or semi-manual data being provided (coming out of a spreadsheet macro, for example) the data is probably going to be extremely inconsistent and opportunities to change producer behaviour probably extremely limited. The perfect solution for this in my mind is a small file upload web UI that overlays a public api, like mentioned previously. This gets the benefits of the api method (real time validation, pushes errors left) without requiring much change in producer behaviour if they are already manually sending the data. Obviously, this represents the most work for you and probably isn't an easy sell to management within your own organization.

For the rest of my suggestions, I'll be assuming you've exhausted all avenues for setting up an alternative process and have no option but to ingest csv data from FTP or email.


## So if you have to...
So assuming, for organizational reasons outside of your control, you have no option but to set up a crappy data file ingestion then here's the principles I think are important to stick to:

### 1. Don't accept excel files
Excel spreadsheets suffer from the same problems as CSVs in this context (no types or schema) but are also much more complex binary files and debugging weird data coming from an excel file is always more difficult. If the data provider can provide an excel spreadsheet, then they should be able to provide a CSV - insist that they do.

### 2. Don't accept not unicode encoded files
Just don't, your strings will thank you later. There really should be no reason to stray from utf-8 for a csv file so push on the data producers on this.

### 3. Formally define your expectations on the data
During the scoping of the project, the data producer will probably send you the infamous "example file". The temptation is to just take this file, plug it in as a fixture in a test, write some code to parse the test files and ingest it and call the job done.

My advice would be, before doing that, take a look at the file (in a text editor, NOT excel/google sheets etc), define formal expectations on the data based on the format they have sent and feed these back to the data producer to get written agreement on these expectations of the data.

When defining these expectations, it can be useful to think about *logical* data types and their *physical* representation. Logical types are the types of data theoretically being represented and their physical representation is the string in the csv representing that type (remember csvs have no types, all values are simply strings of characters).

So don't simply say:
```
We expect theses columns to be present:
    - transaction_id: number
    - start_date: timestamp
    - value: number
```

Say:
```
We expect theses columns to be present:
    - transaction_id: an integer number, e.g. '1'
    - start_date: a timestamp represented in "yyyy-mm-dd HH:MM:SS" format, e.g. '2024-01-01 01:00:00'
    - value: a decimal number represented to two decimal places, zero padded, e.g. '1.20'
```

Don't be tempted to use a library like pandas to automatically infer types for you and hope this will make any inconsistencies in the data just disappear, this approach will work until it doesn't.

### 4. Be strict when expectations are not met
For me this is the rule I see people break the most, to their own detriment.

If the data producers are using an off the shelf tool or a sql script to output the file for you, it may be fairly well automated and therefore consistent in it's output. However, for data produced by excel macros, the format is likely to be very inconsistent to begin with: columns will change name, date formats will change, random whitespace will appear. 

The temptation in this situation is to shake your head, say "god, these guys", and wack in a line of code that tries both column names, iterates over date formats or strips all whitespace. I once had a manager suggest to me that instead of pushing back on the data producer to provide a consistent number of header rows in an excel file they provided to us (an ingestion set up before my time) we should "just iterate down the rows until you find data" with an attitude like I was being stupid for not considering this bulletproof approach.

The problem with doing this kind of thing is that without pushing back, the producers won't realize that it is important that the data stays consistent and they will *always* find new and interesting ways to defy your parser.  The more you chase the dragon of trying to guess the format of data you are receiving, the more complex your code base becomes making it hard to maintain and may become incompatible with older files, making re-loads of data much more of a job than they should be. I'll repeat what I said at the beginning, this data is forming an API between two systems and should be treated as as such. No one would design a web API that attempted to guess what the json payload was supposed to look like. No, a schema is defined, agreed, and deviations receive a 4xx error. For some reason because the technology changes, people seem to forget the principals they apply to everything else.

Some people who work all day in excel find it hard to understand why someone might complain about the column "transaction_id" becoming "TransactionId" the next time data is sent - like it's pretty obvious what it is right? You should make it clear that the data should be provided in a *consistent* format, take the opportunity early after go live / during UAT to cement this by ensuring all deviations are rejected and new data files are requested. Following step three will help you with this by providing a written agreement you can point to when rejecting files. After a bedding in period you should see that the data does start coming through consistently; people are quite good at solving problems when you make them theirs.

### 5. Validate the data immediately
Don't just assume the data will be in the format youve agreed and eagerly try to load / use it. Have a well defined validation step in the ingestion process that aligns tightly with the formal expectations agreed with the producer in step 3. Ideally get validation failures to automatically notify the producers of the data with an informative message about what was wrong with the data (thus providing an automated solution to step 4)

### 6. Convert the data in the same step as you validate it
Say you agree a date column will be coming through in American format (month/day/year... god, these guys), that means your validation process as outline in step 5 will need knowledge of this format to validate that values in this column match it. If you then simply save the file in a raw/staging area the next process that then picks this file up will probably need to know this too, say to insert it into a database as an actual date type. That means you either have to duplicate the knowledge of the file format, or have some central definition that both of them reference. My advice would be to validate and convert the data to proper types in the same step, saving it either as a file with types/schema like Parquet or inserting into a database. The validation and the parsing of the data are somewhat inseparable and nothing can make you more confident that the physical representation of the data can be mapped to its logical types than just doing it - no reason to fight against this.
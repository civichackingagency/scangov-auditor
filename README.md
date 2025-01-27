# ScanGov audit runner

This is a project that has the list of sites to scan in a database, runs scans on a schedule and writes the results to S3 along with logging audit history into to the database.

In addition to the cron audit feature this function will also return:
- List of all domains that were able to be audited
- All domains that were offline last time checked
- All domains that are now redirects to different domains
- All log data for a specific domain

Currently this is running audits for the following

### Web performance

Web performance query is retrieve using this process:

- does CrUX respond with data? 
  - Yes! cool, write to db log, put data on S3
  - If not is site not responding at all or is it a redirect to another domain?
    - If so log this status to db
    - If site is responding but has not CrUX data it is probably a relatively low traffic site so request a lab run lighthouse audit


## Database design

```{
  PK: "DOMAIN#example.com",        // Partition Key
  SK: "METADATA#latest",           // Sort Key
  domain: "example.com",           // Actual domain string
  status: "ONLINE",                // Current status: ['REDIRECTS','ONLINE']
  lastCheckedAt: "2025-01-25",     // ISO date string
  historyLog: {                    // Dates checks ran and links to files
    // ... data log
  }
}

GSI1PK: "STATUS#ONLINE",          // GSI Partition Key
GSI1SK: "2025-01-25",            // GSI Sort Key (lastCheckedAt)
```

### Queries supported

- Query all domains with specific status:

```// Using GSI1
QueryInput = {
  IndexName: "GSI1",
  KeyConditionExpression: "GSI1PK = :status",
  ExpressionAttributeValues: {
    ":status": "STATUS#ONLINE"
  }
}```

- Get information for a single domain:

```// Using base table
GetItemInput = {
  Key: {
    PK: "DOMAIN#example.com",
    SK: "METADATA#latest"
  }
}```

- Get 5 records with old check date and specific status:

```// Using GSI1
QueryInput = {
  IndexName: "GSI1",
  KeyConditionExpression: "GSI1PK = :status AND GSI1SK < :date",
  Limit: 5,
  ExpressionAttributeValues: {
    ":status": "STATUS#ONLINE",
    ":date": "2025-01-25"
  }
}```

## Where is data stored

This Lambda function reviews the sites in the database on a schedule and writes the audit results to a publicly accessible S3 bucket: ```audits.scangov.org```

## Running locally

This uses the <a href="https://openjsf.org/">OpenJS Foundation</a> backed <a href="https://arc.codes/">Architect</a> library which provides a nice wrapper for writing node.js to run on AWS Lambdas.

## Deploying

The AWS credentials are defined in the ```app.arc``` file in the ```@aws``` section:

```
@aws
profile scangov
```

The profile line references the name of a local AWS credentials profile.

To deploy run:

```
npm run deploy:staging
```

or

```
npm run deploy:production
```

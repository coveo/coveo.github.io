---
layout: post

title: "DynamoDB Client Side Encryption"
tags: [Java, DyanmoDB, AWS]

author:
  name: Jonathan Rochette
  bio: Analytics API Jedi Master
  twitter: JoRochette
  image: jrochette.jpg
---

Since the June 20th 2015 release, it is possible to share custom reports with specified users. Before this release, the API calls for managing the custom reports were in the Coveo Cloud Platform API. This was mainly because the Usage Analytics service had no database other than [AWS Redshift](http://aws.amazon.com/redshift/) and we decided that report configurations did not belong in an analytics optimized database. We opted for the Cloud Platform database which is more configuration oriented.

As the Analytics service grows, the need for its own configuration storage began to be felt, so we decided to move the custom reports API to the UA Service and store the configuration in a new database. After an investigation, we decided to use [AWS DynamoDB](http://aws.amazon.com/dynamodb/). Considering that this data can potentially be sensible customer information, it needs to be encrypted.

<!-- more -->

Since server side encryption is not available for DynamoDB, we had no choice but to use client-side encryption. We used a [neat little library from aws-labs](https://github.com/awslabs/aws-dynamodb-encryption-java) for this. It essentially takes care of signing and encrypting the data and is very simple to use.

Here is what the setup looks like. Note that we use [AWS KMS](http://aws.amazon.com/kms/) to manage the encryption key, [Guice](https://github.com/google/guice) for dependency injection and [Archaius](https://github.com/Netflix/archaius) for configuration management :

{% highlight java %}
public class DynamoDB extends AbstractModule
{
    @Override
    protected void configure()
    {
    }

    @Provides
    @Singleton
    public static AmazonDynamoDBClient amazonDynamoDBClient(AWSCredentialsProvider credentialsProvider)
    {
        return new AmazonDynamoDBClient(credentialsProvider);
    }

    @Provides
    @Singleton
    public static DynamoDBMapper dynamoDBMapper(AmazonDynamoDBClient amazonDynamoDBClient,
                                                EncryptionMaterialsProvider encryptionMaterialProvider)
    {
        return new DynamoDBMapper(amazonDynamoDBClient,
                                  DynamoDBMapperConfig.DEFAULT,
                                  new AttributeEncryptor(encryptionMaterialProvider));
    }

    @Provides
    @Singleton
    public static AWSKMS awsKms(AWSCredentialsProvider credentialsProvider)
    {
        return new AWSKMSClient(credentialsProvider);
    }

    @Provides
    @Singleton
    public static EncryptionMaterialsProvider kmsEncryptionMaterialProvider(AWSKMS awsKms,
                                                                            DynamoDBConfigProvider dynamoConfigProvider)
    {
        return new DirectKmsMaterialProvider(awsKms, dynamoConfigurationProvider.getDynamoDBKmsKeyId().getValue());
    }
}
{% endhighlight %}

It's pretty simple to setup right?

Next, here is an example of an object that we save in dynamo and that is encrypted :

{% highlight java %}
@DynamoDBTable(tableName = "UsageAnalytics-Reports")
public class Report
{
    private String id;
    private String displayName;
    private String account;
    private ReportType type;
    private String configuration;
    private boolean isPublic;
    private Set<String> filterIds = Sets.newHashSet();

    public Report()
    {
    }

    @DynamoDBHashKey
    @DynamoDBAutoGeneratedKey
    public String getId()
    {
        return id;
    }

    public void setId(String id)
    {
        this.id = id;
    }

    @DynamoDBAttribute
    public String getDisplayName()
    {
        return displayName;
    }

    public void setDisplayName(String displayName)
    {
        this.displayName = displayName;
    }

    @DynamoDBAttribute
    @DoNotEncrypt
    public String getAccount()
    {
        return account;
    }

    public void setAccount(String account)
    {
        this.account = account;
    }

    @DynamoDBMarshalling(marshallerClass = ReportTypeConverter.class)
    @DoNotEncrypt
    public ReportType getType()
    {
        return type;
    }

    public void setType(ReportType type)
    {
        this.type = type;
    }

    @DynamoDBAttribute
    public String getConfiguration()
    {
        return configuration;
    }

    public void setConfiguration(String configuration)
    {
        this.configuration = configuration;
    }

    @DynamoDBAttribute
    @DoNotTouch
    public boolean isPublic()
    {
        return isPublic;
    }

    public void setPublic(boolean isPublic)
    {
        this.isPublic = isPublic;
    }

    @DynamoDBAttribute
    public Set<String> getFilterIds()
    {
        return filterIds;
    }

    public void setFilterIds(Set<String> filterIds)
    {
        this.filterIds = filterIds;
    }
}
{% endhighlight %}

Also pretty simple. All the job is done with the annotations from the DynamoDB data modeling lib. `@DynamoDBAttribute` indicates which field is persisted in the database. By default, all the attributes that are annotated with `@DynamoDBAttribute` will be encrypted. As you can see, some attributes can be not encrypted (I'll explain why we need this a little bit later).
They are annotated with either the `@DoNotEncrypt` or the `@DoNotTouch` annotations. The attributes with `@DoNotEncrypt` will be signed, but not encrypted and the ones with `@DoNotTouch` will be neither signed nor encrypted. If you are not familiar with the concept of signing and encrypting, I recommend you to take a look at this [stackoverflow answer](http://stackoverflow.com/a/454069/1546324).

That's pretty much it. From now on, saving an object with the `DynamoDBMapper` will encrypt the data before persisting it in DynamoDB.

![image]({{ site.baseurl }}/images/20150706/dynamo_enc.png)

### Tips, limitations and traps to avoid ###

- Using a setup like the one shown in this blog post will require an unlimited key strength. This is configured in the jvm. Here is a stackoverflow answer explaining [how to do it](http://stackoverflow.com/questions/6481627/java-security-illegal-key-size-or-default-parameters).
- Remember when I said wanted some data to not be encrypted? Here's why: scan operations do not work on encrypted fields. This means it's only possible to fetch by ID or by a scan operation on a *non encrypted* field. For us, it is acceptable that some fields are not encrypted. They do not contain sensible information and allow us to perform smarter retrieval of the data.
- It is possible to back up and restore your encrypted data, as long as the encryption metadada is backed up and restored along with it. For big dynamoDB tables, you can use [AWS Data Pipeline](http://aws.amazon.com/datapipeline/). Since our tables are relatively small and data pipeline can be expensive (it uses AWS EMR and must spawn ec2 instances), we use [this script](https://github.com/bchew/dynamodump) in a cron job.
- I haven't tried it yet, but according to AWS support, KMS key rotation is supported by the `aws-dynamodb-encryption` lib.

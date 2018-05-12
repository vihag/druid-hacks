#Resolving Snappy conflicts in Druid Offline Indexing

<p>
<a href="http://druid.io/">Druid</a> is a high-performance, column-oriented, distributed data store.
Druid can be setup on/alongside an HDFS cluster or can be made cloud native using AWS EMRs and AWS S3.

Druid provides extensions to read straight from Hadoop native file formats like Parquet(community-extension)
and Avro(core-extension).

However, you might run into issues like <a href="screenshots/snappyError.png">this</a> where the druid indexer starts complaining about 
being unable to read snappy uncompressed lengths

This usually arises when the process that is writing the snappy compressed files uses a different snappy version than what Druid is using.

In our specific case
Druid version : Druid 0.10.1/ Druid 0.11.0/ Druid 0.12.0
Cloudera version : CDH-5.10.2/ CDH-5.14.2

So, lets start
</p>

<ul>
  <li>
      <h3>Step 1 : Identify that your druid distribution is running on older snappy versions</h3>
          <ul>
              <li>
                the library at extensions/druid-avro-extensions/ has snappy jar in the version range 1.0.
              </li>
              <li>
                the library at hadoop-dependencies/hadoop-client/<hadoop-version> has snappy jar in the version range 1.0.
              </li>
          </ul>
  </li>
  <li>
      <h3>Step 2 : Resolution</h3>
          <ul>
              <li>
                <p>Replace the older versions of snappy jar with the a newer version(matching with the snappy version that created the input files that the indexer is dealing with)</p>
                <p>In our case bumping to version range 1.1. did the trick!</p>
              </li>
  </li>
</ul>


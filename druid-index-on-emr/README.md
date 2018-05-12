#Running Druid Optimally on the cloud

<p>

<a href="http://druid.io/">Druid</a> is a high-performance, column-oriented, distributed data store.
Druid can be setup on/alongside an HDFS cluster or can be made cloud native using AWS EMRs and AWS S3.

Druid indexing can be either real-time(Reading from some fire-hose) or it can be an offline, batch indexing,
most commonly via Hadoop Batch Indexing.

Now, if you already have a big enough Hadoop cluster with Yarn humming along happily on it, you can
readily offload the Druid batch indexing job to the Hadoop cluster, and Druid and Hadoop can live forever,
happily and ever after.

But, what if Druid is all you need, and having another Hadoop cluster just to support Druid batch indexing
is just too darn costly ? 

Fear not, Druid can be ported onto the cloud, very very cheaply, and all the map-reduce indexing tasks that 
are required can be handed of to Amazon EMRs with a slight hack.

Still here ? Read on!

In our specific case
Druid version : Druid 0.10.1/ Druid 0.11.0/ Druid 0.12.0
Amazon version : AMI 5.8

</p>

<ol>
  <li>
      Change the deep storage to Amazon S3
      <ol>
        <li>Include the 'druid-s3-extensions' in the common.runtime.properties</li>
        <li>
            Change the deep storage configurations to S3, like so
            <p>
              <ul>
              <li>druid.storage.type=s3</li>
              <li>druid.storage.bucket=druid-production-bucket</li>
              <li>druid.storage.baseKey=druid/segments</li>
              <li>druid.s3.accessKey=<your access key></li>
              <li>druid.s3.secretKey=<your secret key></li>
              </ul>
            </p>
        </li>
        <li>
            Optionally, you might want to change the log storage to S3 as well, like so
            <p>
            <ul>
              <li>druid.indexer.logs.type=s3</li>
              <li>druid.indexer.logs.s3Bucket=druid-production-bucket</li>
              <li>druid.indexer.logs.s3Prefix=druid/indexing-logs</li>
            </ul>
            </p>
        </li>
      </ol>   
  </li>
  <li>
        Change the meta-data storage from default derby to postgres on Amazon RDS
        <ol>
          <li>Include the 'postgresql-metadata-storage' in the common.runtime.properties</li>
          <li>Setup the RDS instance with the instructions <a href="http://druid.io/docs/latest/development/extensions-core/postgresql.html">here</a></li>
          <li>Ensure that the RDS instance is allowed through the security groups for your Druid cluster, specifically port 5432</li>
          <li>
              Change the meta data configurations to use RDS, like so
              <p>
              <ul>
                <li>druid.metadata.storage.type=postgresql</li>
                <li>druid.metadata.storage.connector.connectURI=jdbc:postgresql://RDS-END-POINT:5432/druid</li>
                <li>druid.metadata.storage.connector.user=user</li>
                <li>druid.metadata.storage.connector.password=password</li>
              </ul>
              </p>
          </li>
        </ol>   
  </li>
    <li>
          Change the middle manager configuration to work with Amazon EMR
          <ol>
            <li>
              <p>Edit property 'druid.indexer.task.hadoopWorkingPath' to point to a persistent HDFS(can be a small setup on your Druid cluster), or a remote FS like so - hdfs://<name-node>:8020/var/druid/hadoop-tmp'</p>
              <p>
              <i>The reason for this change is that, when the hadoop indexing happens, Druid will write information on the temporary path, regarding segment meta data, locations etc</i>
              <i>If we do not specify a remote path to write to(in this case an IP on the Druid cluster), then the EMR will write this information onto its own HDFS and Druid will not be able to read it post indexing, and hence fail</i>
              </p>
            </li>
            <li>
              Change your indexing specs(the jobProperties section of the json) to override YARN discovery at index-time, like so,
              <p>
              <ul>
                <li>"fs.defaultFS": "hdfs://__EMR_INTERNAL_DNS__:8020",</li>
                <li>"mapreduce.jobhistory.address": "__EMR_INTERNAL_DNS__:10020",</li>
                <li>"mapreduce.jobhistory.webapp.address": "__EMR_INTERNAL_DNS__:19888",</li>
                <li>"yarn.web-proxy.address": "__EMR_INTERNAL_DNS__:20888",</li>
                <li>"yarn.resourcemanager.resource-tracker.address": "__EMR_INTERNAL_DNS__:8025",</li>
                <li>"yarn.resourcemanager.address": "__EMR_INTERNAL_DNS__:8032",</li>
                <li>"yarn.resourcemanager.scheduler.address": "__EMR_INTERNAL_DNS__:8030",</li>
                </ul>
              </p> 
              <p>If you are wondering how to get the EMRs internal DNS, look <a href="https://github.com/vihag/cli-hacks/blob/master/aws_cli/GetEMRClusterMasterDNS.sh">here</a></p>
              <p>If you are wondering how to replace the newly minted EMRs DNS in your indexing spec, look <a href="https://github.com/vihag/cli-hacks/blob/master/bash_cli/ReplaceStringInPlaceInFile.sh">here</a></p>  
            </li>
            <li>Ensure that the EMR has access to the Druid Cluster, and vice versa, like so 
                <p>
                  For example, if your spin up EMRs with the following security group configurations
                  <ul>
                    <li>"EmrManagedSlaveSecurityGroup":"sg-36d8c451"</li>
                    <li>"EmrManagedMasterSecurityGroup":"sg-14d9c573"</li>
                  </ul>
                  Then,
                  the VPC hosting Druid should enable TCP access to sg-36d8c451 and sg-14d9c573
                </p>
            </li>
            <li>
              Useful tip - enable debugging on the EMR atleast for a few initial runs, so that accessing logs for failed YARN jobs is feasible
            </li>
          </ol>
    </li>
  
</ol>

And that's all!
With the deep storage secured on Amazon S3, and Map Reduce secured on Amazon EMR, you are free to keep your Druid Cluster to a minimum size, with space enough to hold segment-caches,
and what's better - all those compute resources can be devoted to Historicals and Brokers for shaving of those last milliseconds on your queries!


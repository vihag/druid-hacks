#Xerces Hell with Druid on Cloudera 5.10.2(and many more)

<p>
<a href="http://druid.io/">Druid</a> is a high-performance, column-oriented, distributed data store.
Druid can be setup on/alongside an HDFS cluster or can be made cloud native using AWS EMRs and AWS S3.

Most users will use a Cloudera or Hortonworks Hadoop distribution to get started quickly with a managed Hadoop Cluster.
This however brings hidden landmines when getting Druid to play with a managed Hadoop distribution.

In this how-to we shall walk through something called 'xerces-hell', which is a well known troublemaker
in the Java world, and can pop up when getting Druid to work on Cloudera.

In our specific case
Druid version : Druid 0.10.1/ Druid 0.11.0/ Druid 0.12.0
Cloudera version : CDH-5.10.2/ CDH-5.14.2

So, lets start
</p>

<ul>
  <li>
      <h3>Step 1 : Identify that you are in Xerces Hell</h3>
          <ul>
              <li>
                <p>The indexing jobs are dying mysteriously with unhelpful error codes showing in the indexer-logs as seen in
                screenshots/indexerLog.png </p>
              </li>
              <li>
                <p>Open the corresponding Yarn log, and there should be a Xerces related stack trace as seen in
                screenshots/signatureConflict.png</p>
              </li>
              <li>
                Welcome to xerces hell, it's a wonderful place ;)
              </li>
          </ul>
  </li>
  <li>
      <h3>Step 2 : Confirm xerces conflicts between druid hadoop dependencies and cloudera</h3>
          <ul>
              <li>
                First off, it is a good idea to `pull-deps` the correct version of cloudera hadoop dependency for druid like so
                            `java -classpath "druid-0.12.0/lib/*" io.druid.cli.Main tools pull-deps -r https://repository.cloudera.com/content/repositories/releases/ --clean -h org.apache.hadoop:hadoop-client:2.6.0-cdh5.14.2`
              </li>
              <li>
                Check the versions of xml-api and xercesImpl jars in your Druid installation
                 <ol>
                    <li>druid-0.10.1/hadoop-dependencies/hadoop-client/2.6.0-cdh5.14.2/</li>
                    <li>druid-0.10.1/extensions/druid-hdfs-storage</li>
                 </ol>
                 At the time of this publication, they are 
                  <ol>
                    <li>xercesImpl-2.9.1.jar</li>
                    <li>xml-apis-1.3.04.jar</li>
                  </ol>
                 And here-in lies the heart of the conflict 
              </li>
              <li>
                Check the versions of xml-api and xercesImpl jars in your Cloudera installation
                The jars are usually located at `/opt/cloudera/parcels/CDH/jars/`
                At the time of this publication, they are
                <ol>
                  <li>xercesImpl-2.10.0.jar</li>
                  <li>xercesImpl-2.9.1.jar</li>
                  <li>xml-apis-1.3.04.jar</li>
                  <li>xml-apis-1.4.01.jar></li>
                <ol>
                As you can see there are two versions of xerces available(xml-apis are the interface collection for xerces, don't get confused by the name).
                And this can potenially trip up the Hadoop Classloader
              </li>
          </ul>
  </li>
  <li>
    <h3>Step 3 : Resolve the conflict</h3>
    <ul>
      <li>
          <p>
          In the two conflicting libraries in Druid(druid-hdfs-storage/ & hadoop-dependencies/hadoop-client/<version>/);
          Replace xercesImpl-2.9.1.jar with xercesImpl-2.10.0.jar
          and
          Replace xml-apis-1.3.04.jar with xml-apis-1.4.01.jar>
          </p>
      </li>
      <li>
          For good meaure, edit your Yarn classpath at either 
          <ul>
            <li>the client configuration you have in the druid installation at `druid/conf/druid/_common/yarn-site.xml`</li>
            <li>the cloudera admin console(depending on your access level)</li>
          </ul> 
          to include xercesImpl-2.10.0.jar specifically.  
      </li>
   </ul>
   And this should do it!!
  </li>
</ul>


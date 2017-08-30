# Speaker notes!

## About me
Shit, writing about me is like the hardest part of any talk.

No one really cares about your hobbies or whatever.
I like to write software?

###
When I first started at cornell, our home page was 10 MB of downloads on mobile devices.  It's now below 2 MB.

We had one guy kind of using git.  Now, we self host Gitlab.

And I figured out that when you run sudo on a CIT server, the incident is actually logged and will be investigated.  And that you'll have to explain that to your boss why he got a voice mail from CIT.

## Univcomm

I work for university communications.  If you haven't heard of us, we run Cornell.edu, Cuinfo, Cornell Cast, NYC, and many other sites around the university.  If you've ever used Around the University, that's us too.  We've got over 20 internal apps that are used by many departments around campus.  The team is 2 designers, 4 developers; and I kind of defaulted to the ops guy even though my title is still developer.

While cornell.edu isn't the highest traffic site at cornell, nor the most important, its one of the most noticeable when it fails.  In an average month, we get between 25,000 and 180,000 requests per hour.  In terms of pageviews, it's 50,000 pageviews per day with between 3000 and 4500 per hour during work hours.  In terms of requests per second, a typical high is around 380, and lows are, of course, 0.  Recently, we've been the target of someone's benchmark and logs indicate that we can handle 1200 r/s without stutterring.

## Zero

Right now, we've got 4 "massive" machines serving cornell.edu.  They're dual processor systems with 4 GB of ram each.  Each one of these servers can handle hundreds of requests per second.  In gathering stats for this talk, I saw that we were getting 1400 requests at once from a benchmark tool every morning and it did not make a noticable impact.  Cornell.edu was written to be heavily cached.  We haven't tested because it's a production system, but my guess is apache's max servers would be the limiting factor before the size of the servers.  While these servers can handle 300 r/s, they usually idle below 7r/s.

We've also got two even larger servers for our backend processes, admin tasks, and some of our more processing intensive sites.  

Our software stack pre-dates dirt.  It's been reliable, but of all the server languages we have, they're all past end of life.  The only thing that's actually not EOL is Apache 2.2.  Maintenance updates ended in July.  Security updates may end in December.

Our machines mount NFS folders as their webroot.  Each machine has it's own vhost config, which we have partial access to.  Each NFS folder is roughly an "instance".  Instances are spread across several servers.  So, we have 6 servers that mount cornell.edu's instance, 8 servers that mount media.univcomm, and 4 servers that mount apps.  In production, only 4 machines serve cornell.edu.  The extra two that mount it are our dev machines.

We have ssh access on our two dev instances, but for the most part, file access is done through webdav.  Webdav is logged, so if someone breaks prod, you can check the logs and find out who.  But it's also slow and unreliable.  What gets us by is for any site we need to update, we have 4 machines that we can access the files through.  

## History

This is zero to autoscaling from an apps developer POV.  If your team doesn't have good process and strong ops skill already, it's going to take work.  We're still in process, honestly.

So, here's what you've missed since we started down this road.  The day I got here, we had one developer using git through webdav.  As I said, webdav is slow.  Git through webdav is painful.  And we have crashed prod using git over webdav.  My second month here, we had a migration from CF9 to CF10.  We survived, just barely.  I remember my boss calling me saying "hey, prod is down and I can't get anyone else, can you do anything?"  But things got better and by month 3, I had a docker container running apache and coldfusion and started sharing it with my team.
By month 6, we were working with CIT to figure out how to bypass webdav so we could use git for deployment.  This was probably the point where we could really say we used git.  By month 12, we had outgrown our paid github instance splitting our our projects into logical units.  We evaluated bitbucket for EDU and Github other options.  Because we've got 9 users who need access to our repos, Bitbucket wasn't free.  Github was going to be expensive.  And we found that we had 2 rackspace servers already doing nothing.  So, we started running gitlab on rackspace as a managed application.

Two months later, we enrolled in AWS through the Cloudification team.  A month later, I did an emergency migration from rackspace to AWS after I did something stupid.  Check your backups, because that's what saved us.

The docker and local development approach didn't sit well with some of our devs, so we started using ansible to build dev servers on AWS.  This let us use real domains, ssl, CU Web Auth, etc.

After the AWS hackathon, we deployed our first lambda function, then another.

It's about month 27 of my timeline.  We've used s3's static site hosting (right around the time of the s3 outage), we've got a couple lambda functions in prod, a couple of sites on continuous integration.  Gitlab is on autopilot and takes almost no time for management.  We're still constrained by what is technically possible.  Our portfolio is too large to rewrite all at once and either way, we don't have a target.  Are we doing CF11, CF 2016? Lucee? Railo?  Rails?  Either way, we need to figure it out.  And we need to be able to do it parts at a time.

# Failures

I've been thinking about this for a while.  We've tried several ways to get our apps running in the cloud.  One of the more promising ones was running coldfusion on plain old tomcat.  Elastic Beanstalk can run Tomcat directly.  Coldfusion runs Tomcat.  Can I turn Coldfusion apps into Java apps and stop thinking about Coldfusion?  AWS EB understands WAR.  Coldfusion can make WARs.

I'll save you the effort.  Adobe modified Tomcat rather than make Coldfusion act reasonably.  So when you give a coldfusion war to tomcat, it doesn't run.  If you dig deep, you'll find the SES urls are the main culprit.  The good news is you can actually get around it.   Coldfusion is just another java app that runs on tomcat.  WARS are just fancy zips.  WAR files map urls to servlets through web.xml.  You can disable SES and a number of other servlets including cfadmin through web.xml.  Further, if you've been exporting your wars through cfadmin, once you rewrite the files, further exports won't revert your changes.  The export process is just an ANT build, so you can actually create everything with cfadmin and automate it later with the correct CLI tools.

A couple things to note if you're going to persue this method.  There's CFClasspath and Java classpath and they're different.  Your cf classpath wants to be in the war file itself, which will ad 50MB to your deployable artifact.  The config that's normally handled by CFAdmin will now be handled by tomcat; by that, I mean that your database passwords will be encrypted in the war file, but the decryption key is with them.  If someone gets your artifact, they have your DB credentials.

One cool thing I realized in preparing for today; Elastic beanstalk's tomcat runs apache as an RP, so in theory, you can add CUWA using eb extensions.

The take away for me was Ansible maps really well to a fleet of long lived pet servers.  Things you name and care for and replace when they die.  There is a way to have it manage dynamic inventory.  I didn't get there.  I still want to look at running our apps directly on an amazon managed tomcat server.  It'd be really easy to pawn it off on them to keep tomcat, apache, and whatever version of linux they use up to date.  Less work for my team, less work for me. #shipit

## Renewed vigor

With renewed vigor I went back to my roots.  I started looking at things like bus factor, "minimum viable product", and GSD.  And I realized we're a long way from prod, so we need to get there first.  With that in mind, upgrade what you can, keep what you have to, and make the parts replacable, mockable, and at least somewhat testable.  Thusly Cloudfusion was born.

Cloudfusion is not a thing you can point git at and say clone.  Cloudfusion is a combination of Docker containers, a docker trusted registry, Elastic Beanstalk, Gitlab, and a CI/CD approach.  It's not perfection.  It's good enough to push forward and that's about it.

Following the idea of GSD, and to avoid having such a low bus factor, Cloudfusion uses one container running Apache2.4, Coldfusion 10 fully updated, java 8, and postfix because our apps expect to be able to send email.  Because unix, and init systems, and "I know this works", cloudfusion uses phusion baseimage.  The container is built to fetch secrets that it needs using AWS credentials.  

_Elaborate on this_

Elastic beanstalk handles rolling code updates, auto scaling, load balancing, log collection, etc.  We use EB extensions to configure our NFS, bring logs out from the container to the host, set up log rotation, swap space on smaller servers, and set up the health daemon for the load balancer.  As a bonus, you can use EB to run the container on your local machine, which is one step closer to "pull repository, run command, get to work" for our team.

We've centralized code and CI into Gitlab.  Our container build process pushes to DTR.  EB deploys pull from DTR.  Pushes to our application repositories deploy to EB.  We're a few steps off of blue/green deployment, but it's not far.  

It would be absolutely insane to do this in production, but, we currently have no testing.  And we don't know what's happening in the future with regard to languages, so an investment in testing also wouldn't get a tonne of support.  So, I hooked a gatling's test recorder to a site spider I wrote using ruby and phantom JS.  What came out is a smoke test.  Run once against your production site, change the base url, and run it against the new site.  It will expect all the same resources to be there.  The one issue is I need to make the test fail on non-200 status for any request.

# Still left

Like I said, we're not production ready.  We're working with CIT to get a splunk instance in AWS.  We still need to figure out how we'll move our databases or if we will.  Cornell.edu is not very intense with database calls, so we could get away with on-prem databases in the short run.  Even if we don't move services to AWS, we need to move to a non-EOL version of coldfusion.  If we do move, we need to consider how we will handle production SSL certificates and production keytabs for CUWA.  My guess is there's going to be more red tape moving www than .. I don't know, something that has a lot of red tape. One of those old time stock tickers from a really important company the day of the market crash?






## Important links

### Get into media volume before apps
http://media.cuweb.tech/photos/400x225/30A99863-9933-EB0E-5C8B981212D88945.jpg

### apps
http://apps.cuweb.tech/updater/

### cuinfo
http://cuinfo.cuweb.tech


https://twitter.com/sadserver/status/895002207313158144

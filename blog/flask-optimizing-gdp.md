# Preamble

In TU Berlin every computer science student has to complete one "practical" project in order to graduate. From the University there are 5 different faculties and each
offers one project. Theoretically you can propose your own project and present something from your work, but practically it means double efforts, more risks and a way 
more bureaucracy.

I picked project from Telecommunication Networks Group ("TKN") as it stood out from other faculties - instead of "divide students into groups, each group makes their 
own project and then we compare them with each other"-approach, we were separated into groups, each working on a part of a common project. This meant that instead of 
usual communication within group of 5 students we also had to communicate with 3 other groups each doing something on their own. Without any oversight or management 
but with way too much of enthusiastic students (most of which had no real-world working experience) this actually turned out to be quite problemmatic. 

The initial goal was to create a system, which would track and predict movements and activities of a user using his phone, smart watches and some "intelligent" home 
sensors. Our group was working on a REST-server and we had to glue together groups working on sensors, machine learning an data visualisation. The only condition from 
professor was that we had to use [GDP](https://gdp.cs.berkeley.edu/redmine/) for storing log data isntead of any other database.

# Situation

After we've dealt with "how exactly should our REST interface look like" and "which language should we use" kind of arguments (picked flask as it suited best poor GDPs 
language bindings) and managed to deploy our server on the AWS EC2 server we've encountered our first serious problem. GDP is a "single process per system" kind of
software and it does not support concurrency and parallel execution in any way. So, a tool which we had to use for a web service because "it has publish/subscribe 
feature" could not handle at very least "subscribe to one log and then read data from other log" use case. Since we found no decent library for events on python 2.7 
(which we had to use because of GDP) we've improvised and created our own internal log for events, so that the other groups can still subscribe to multiple logs. 

Also, this decision killed any possibility for horizontal scaling - we had to assume all the updates are going through our single instance of server, otherwise log 
updadates would be missed. 

Then on one of milestone presentation we've realized that visualization group is not using javascript EventSource to recieve notifications "on-the-fly" - instead 
they are sending 2 GET-requests per second to get last event from 2 different logs. And the sensor data group is posting at least 1 requests per second. GDP is 
blocking every thread, executes one request at a time, and also it stores its logs somewhere in California so we had to add latency for every request. It takes
way more than a second to proceed every request, and in that time new requests are comming and starting to pile up... 

### Congratulations, our server is dead and it took only 2 users to achieve that. 

# First step - obtaining metrics


# Pipe lab

This lab will introduce you the Unix `|` (pronounced "pipe") operator.
You will use the pipe to do basic analysis of a large twitter dataset.

(There is no need to work with a partner on this lab,
but you are of course encouraged to do so.)

## Part 0: Measuring the Dataset

The directory `/data/Twitter dataset` contains a large dataset of tweets.
Run the following command to see the size of the dataset
```
$ du -h '/data/Twitter dataset/'
```
The `T` in the output stands for [terabytes](https://en.wikipedia.org/wiki/Byte#Multiple-byte_units), which equals 1000 gigabytes, or $10^9$ bytes.
Over the next few labs and homeworks,
we will practice techniques for working with these large datasets.

> **NOTE:**
> Notice that the path `/data/Twitter dataset` contains a space inside of it.
> That is why the command above has quotation marks around the path.
> Also notice that when I put the path in an inline code segment,
> I do not use quotation marks.
> `'/data/Twitter dataset'` is not the path, because the path does not have quotation marks inside of it.

## Part 1: Counting the Files

The first step in getting familiar with a new dataset is just to count how many files there are.
We can easily do that by combining our knowledge of the `ls`, `wc`, and `|` commands:
```
$ ls '/data/Twitter dataset' | wc -l
```
Importantly, however, not all of these files contain actual data.
For example, there is a file `/data/Twitter dataset/readme.txt` that describes the format of the dataset.

There is another `.txt` file in the dataset folder.
How can we easily find the name of this file?
We can combine the `ls` and `grep` commands like so:
```
$ ls '/data/Twitter dataset' | grep 'txt$'
```
Recall that `grep` takes a regular expression as its argument,
and outputs only those lines that match the regular expression.
In this case, the `$` indicates the end of a line,
and so `grep` will only print those files that end with the letters `txt`.

If you read through these files, you will discover that:
The data is partitioned into zip files,
where each zip file corresponds to a single day of tweets.
Our next task is to discover how many days worth of data the dataset contains.

Once again, we can answer this question easily with our shell commands.
In this case, we'll need to use `grep` to filter the output of `ls` to keep only files that end in `zip`, then use `wc` to count the results.

**Problem 1:**
Write a one line shell command that returns the number of days worth of data (i.e. the number of zip files) in the twitter dataset.
There is no need to turn anything in for this problem,
and you can find a solution in the comments of this README.
<!--
$ ls '/data/Twitter dataset' | grep 'zip$' | wc -l
Your command doesn't have to exactly match that, but you should get 1887 as your answer.
-->

## Part 2: Counting the Tweets

Our next task is to count how many tweets the dataset has per day,
and once again, we can easily do this with the shell.
The file `/data/Twitter dataset/geoTwitter20-01-01.zip` contains the tweets sent on 01 January 2020,
and we can use the `unzip` command to extract its contents.
Run the command:
```
$ unzip '/data/Twitter dataset/geoTwitter20-01-01.zip'
```
After a few seconds, you should get an error message that looks something like
```
write error (disk full?).  Continue? (y/n/^C)
```
By default, the `unzip` command unzips the contents of the zip file to your current working folder.
You only have 100MB of disk space allocated,
and the file contains about 6GB of data,
so there's no way for you to save the decompressed contents to the hard drive.
Free up your disk space by using the `ls` command to identify the name of the folder created by the `unzip` command, and the `rm` command to remove this folder.

To work with this zip file, we will need a technique that uses only $O(1)$ disk space.
The answer is to have `unzip` print the contents of the file to the screen rather than saving them to the hard drive.
We can do this with the `-p` flag:
```
$ unzip -p '/data/Twitter dataset/geoTwitter20-01-01.zip'
```
If you run the command above, you will see a LOT of information printed to the screen,
and it will take more than an hour to finish.
Stop the command by pressing CTRL-C on your keyboard.

For commands with a lot of output, it is common to want to inspect only the first few lines of the output.
The `head` command lets us do just that.
Print only the first line of the zip file with the command
```
$ unzip -p '/data/Twitter dataset/geoTwitter20-01-01.zip' | head -n1
```
The first line of the zip file is rather long, so it will still probably fill up your terminal screen,
but the command will at least run quickly.

The format of this zip file is that each line is a JSON (pronounced like the name "Jason") object which contains all the information about a tweet.

We'll take a detailed look at the contents of the JSON object in the next section, but for now, the important thing is that each line of the zip file contains exactly one tweet.
This lets us count the number of tweets sent on a particular day using our standard pipe tools:
```
$ unzip -p '/data/Twitter dataset/geoTwitter20-01-01.zip' | wc -l
```
Unfortunately, the file is so large that just looping over it and counting the number of lines takes a long time.
For me, it takes about 80 seconds when no one else is on the lambda server.
With other people on the lambda server trying to access the same file,
this could take a few minutes.
You don't have to let this command run until completion,
and you can press CTRL-C to end it early.

> **ASIDE:**
> If you let the command above run to completion, you'd see that there are about 4 million (or exactly 3930907) tweets in the file.
> These are ALL of the tweets where users have enabled "geolocation" information.
> About 2% of twitter users enable geolocation, and so this is a snapshot of the 80 million tweets that were sent on 01 January 2020.

> **ASIDE:**
> It is useful to time long running commands,
> and the shell has a built-in command `time` for doing this.
> If you re-write the command above as
> ```
> $ time (unzip -p '/data/Twitter dataset/geoTwitter20-01-01.zip' | wc -l)
> ```
> then the terminal will print the runtime for you.

Let's assume for a second that the best-case runtime for looping over one of these zip files is 80 seconds.
How long would it take to loop over the entire dataset?
We can use the `|` operator combined with the `echo` and `python3` commands to calculate the total number of hours:
```
$ echo "print(1887*80/60/60)" | python3
41.93333333333333
```
Based on this output, we can predict that it will take about 42 hours just to loop over the dataset!
Because looping has a runtime of $\Theta(n)$,
we can estimate that an algorithm with runtime $\Theta(n^2)$ would take time $42*42=1764$ hours, or 73 days to complete.
Therefore, whatever algorithms we perform on this dataset,
we'll want them to have runtime $O(n)$.

> **NOTE:**
> I was careful with my big-O notation above.
> I switched from $\Theta$ to $O$ because it would be okay to have a runtime that was sub-linear and take less than 42 hours.
> They key idea is just that we don't want anything super-linear since that will result in absurd runtimes.
> Big-O lets us express both of these ideas simultaneously in a compact notation.

## Part 3: Inspecting Tweets

In the last task, we saw how to extract an individual tweet using the `head` command:
```
$ unzip -p '/data/Twitter dataset/geoTwitter20-01-01.zip' | head -n1
```
The output, however, is hard to read.
In order to save disk space, whitespace has been removed from the tweet's JSON object.
Fortunately, python has a built in module for "pretty printing" JSON called `json.tool`.
We can access it by piping the output of the above command into the python module:
```
$ unzip -p '/data/Twitter dataset/geoTwitter20-01-01.zip' | head -n1 | python3 -m json.tool
```
The output looks like a python dictionary.

> **ASIDE:**
> Technically the output you're looking at is a JavaScript dictionary.
> (The J in JSON stands for JavaScript.)
> Conveniently, however, JavaScript and Python have very similar syntaxes,
> and every valid JSON object is a valid Python object as well.

Find the `"text"` key of the JSON dictionary.
(Notice that I referred to the key as `"text"` and not `text`.
The content of the string is `text`, but the value of the key is the string literal `"text"`.)
This is the actual text of the tweet, and you should see that it is equal to
```
🤘🏾 @ Pappadeaux Seafood Kitchen https://t.co/1oecbjJmP2
```
These tweets are sorted based on the time they were sent,
and so we can see that the very first tweet of 2020 was an ad for a seafood restaurant :/

To see who sent this tweet, look at the `"user"` key.
The value for this key is another dictionary containing all the information about the user.
Inside this dictionary, you'll find another key `"screen_name"`,
which shows that @SeanFalyon sent the tweet.

Recall that all of these tweets are geolocated.
That means they contain the location that the twitter user sent them from.
This information is located in the `"place"` key,
which contains a dictionary describing the place the tweet was sent from.
Inside this dictionary is a value `"full_name"` that contains the name of the city and state that the tweet was sent from.
In this case, the tweet was sent from Houston, TX.

## Part 4: `jq`

The terminal has a useful command `jq` for extracting information out of JSON objects.
(`j` is for JSON, and `q` is for query.)
Run the following sequence of commands:
```
$ unzip -p '/data/Twitter dataset/geoTwitter20-01-01.zip' | head -n1 | jq '.["text"]'
$ unzip -p '/data/Twitter dataset/geoTwitter20-01-01.zip' | head -n1 | jq '.["user"]["screen_name"]'
$ unzip -p '/data/Twitter dataset/geoTwitter20-01-01.zip' | head -n1 | jq '.["place"]["full_name"]'
```
You should see the values of the keys we discussed in the previous section above.

> **ASIDE:**
> The argument to `jq` is technically called a JsonPath.
> JsonPath is a standard for extracting information from JSON objects,
> and like all modern standards,
> the details are stored in a git repo: <https://github.com/json-path/JsonPath>.
> The documentation for the `jq` command is [also hosted on github](https://stedolan.github.io/jq/tutorial/),
> this time in the form of a standard HTML webpage hosted using [github pages](https://pages.github.com/).
> (For those of you who took CS40 with me, recall that github pages was how you created your webpages for that class.)
> The JsonPath syntax can perform incredibly complex queries,
> but you don't need to know the details of this syntax for this class.
> Hopefully, the simple queries above are relatively easy to understand, though.

Lets see one more example using the `jq` command to process multiple tweets at once.
Recall that the `head -n1` command outputs only the first line of its input and discards everything else.
We can change the number `1` to a larger number to get more lines,
and the `jq` command will then process each of those lines individually.
The following command prints the text of the first 20 tweets sent in 2020:
```
$ unzip -p '/data/Twitter dataset/geoTwitter20-01-01.zip' | head -n20 | jq '.["text"]'
```
Notice that many of these tweets are in non-English languages.
(Only about 30% of tweets are English.)

**Problem:**
Write a one line shell command that returns the name of the location that the first 10 tweets were sent from on 1 April 2020.
There is no need to turn anything in for this problem,
and you can find a solution in the comments of this README.
<!--
$ unzip -p '/data/Twitter dataset/geoTwitter20-03-01.zip' | head -n10 | jq '.["place"]["full_name"]'
-->


# Part 5: Interesting queries

We're now ready to perform some rather interesting queries on our dataset.
For example, the following query will list the text of all tweets sent from Claremont, CA on new years day 2020:
```
$ unzip -p '/data/Twitter dataset/geoTwitter20-01-01.zip' | grep "Claremont, CA" | jq '.["text"]'
```
> **NOTE:**
> You don't need to use the `jq` command to filter a tweet with the `grep` command because `grep` will check all of the fields in the JSON object.
> (Make sure you understand why!)
> The `jq` command is most useful for just changing how things get printed.
> Technically, the command above will also catch tweets that contain the string `Claremont, CA` anywhere in their JSON object.
> This could happen, for example, if a person tweets: "I'm in Claremont, CA right now."
> A more technically correct regex would be `"full_name":"Claremont, CA"`,
> but using this more correct regex wouldn't actually change the results of your answer in this case,
> so I've opted for the simpler regex above.

And the following command will count the number of times that "coronavirus was mentioned on 01 April 2020:
```
$ unzip -p '/data/Twitter dataset/geoTwitter20-04-01.zip' | grep -i "coronavirus" | wc -l
```
> **NOTE:**
> The `-i` flag to `grep` indicates that the regular expression is case insensitive.
> So this regex will match examples like `coronavirus`, `Coronavirus`, `CORONAVIRUS`, and `CoRoNaViRuS`.
> It will also match lines no matter where in the line the text appears,
so it's fine if the text appears inside of a hashtag.
> Also, I'm not using the `jq` command because I'm just counting the results.

## Submission

Write a command that outputs the text of all of the tweets sent from Claremont, CA on 12 March 2020 that mention the coronavirus.
(This was the last day of in-person classes that semester.)
Upload your command and it's output to sakai.


<!--
> **ASIDE:**
> When you press CTRL-C, you will see the character sequence `^C` printed to the terminal.
> The `^` on the terminal is how the CTRL key is represented,
> and so people will sometimes write `^C` instead of CTRL-C.
> Technically, what's happening when you press CTRL-C is that your terminal is sending a special ASCII character to the shell called the [End of Text (ETX)](https://en.wikipedia.org/wiki/End-of-Text_character).
> This is a non-printable character in the ASCII table that doesn't have a key on the keyboard,
> but the CTRL key lets you send non-printable ASCII characters.
> The shell running on the lambda server is never away that you've pressed the CTRL key.
> This is called [caret notation](https://en.wikipedia.org/wiki/Caret_notation).
> ```
> The BST assignment totally sucks^h^h^h^h^hrules.
> ```
>
> The CTRL-h command is equivalent to the backspace character.
> (Very old keyboards didn't have a backspace character,
> The wikipedia article on [caret notation]
-->


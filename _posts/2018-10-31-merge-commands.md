---
layout: post
title:  "Checkbox-cli new sub-commands"
date:   2018-10-31 12:18:20 +0200
categories: test ubuntu
tags: cli
---

Time to introduce three new amazing commands now available with `checkbox-cli`. 

- `merge-reports`: Create a single QA HTML report from multiple 
[SKU](https://en.wikipedia.org/wiki/Stock_keeping_unit) results.
- `merge-submissions`: Combine several submissions tarballs into a new 
submission.
- `tp-export`: Export a test plan to XLSX/PDF.

Stellar? ... yes!

# Some background 

For years the only way to save checkbox session results was exporting to XML. 
It was a nightmare, bringing dependencies on libxml, having to base64 binary 
attachments. Moreover how to share this kind of report with customers. Nobody 
fluently reads XML. Then we introduced both HTML and XLSX exporters as 
companions of the legacy XML format. The first version of the HTML report was 
just XML+XSLT.

With checkbox-ng, a new class of units was created, exporter units. Today's 
HTML report is created from a Jinja2 template, so powerful. We also said 
goodbye to the XML format to generate a submission tarballs containing all kind 
of reports, HTML, XLSX, JSON with attachments and I/O logs in dedicated folders 
within the archive.

But even if those final exports look beautiful and flexible from a developer 
point of view they are not perfect for QA people who still have to manually edit
them to produce a full report. So what's missing?

# merge-reports

Some projects often come with multiple SKU to test. Test plan is the same but 
results depend on the SKU specific features (e.g one can have extra ports, 
another a modem). In that case checkbox runs X times producing X tarballs that 
have to be combined to make a QA report, a customer facing document.

The solution is a single command where the only prerequisite is to have 
sessions with titles. Just run checkbox with `--title "foo"`, the session will 
record it. Then run checkbox as follow to create a new [all-in-one HTML 
report]({{ site.url }}/assets/files/multi-sku.html){:target="_blank"} with a 
drop-down list of the tested SKU:

{% highlight console %}
$ checkbox-cli merge-reports ./foo.tar.xz ./bar.tar.xz ./baz.tar.xz -o /tmp/multi-sku.html
{% endhighlight %}

![merge-reports](/assets/images/merge-reports.png)

# merge-submissions

Second use case is related to the test plan itself and the nature/length of 
each test. It's faster to first go over all manual tests then let the system 
run all automated/stress tests in a separate run. But all results have to be 
combined into a single report for review.

There's a method now to run multiple tests (e.g. a main test run, a network 
retest, stress tests) and then merge the three into a single submission.

{% highlight console %}
$ checkbox-cli merge-submissions ./foo.tar.xz ./bar.tar.xz ./baz.tar.xz -o ./submission.tar.xz --title "Full"
{% endhighlight %}

The new archive contains the full collections of test results, I/O logs and 
attachments coming from the different submissions (you can pass more than 
three). Conflicts resolution is quite simple, I keep the results from the last 
submission passed to the command line, so the order matters.

This [HTML report]({{ site.url }}/assets/files/multi-submissions.html){:target="_blank"} for 
example is the one created when combining the 3 session previously used.

The optional `--title "Full"` will give your new session a title (by default 
the last session on the command line will be preserved).

# tp-export

This command can be used to export any test plan to a spreadsheet document. To 
share an initial plan with a customer or help identifying the missing items for 
a new project.

Test plans often rely on [template jobs](https://checkbox.readthedocs.io/en/latest/units/template.html){:target="_blank"}
making quite hard such export on a system where hardware device properties are
used to generate the new jobs. But `tp-export` runs in a special mode where all 
template jobs are using fake resources. This way the export can be executed on 
a different system. The only exception is the fake resource for GPU. Instead of 
just creating one object per resource (one modem, one disk, ...) it creates two 
instances to simulate hybrid graphics.

Example with the 18.04 certification test plan for Desktop:

{% highlight console %}
$ checkbox-cli tp-export com.canonical.certification::client-cert-18-04
{% endhighlight %}

The command creates a [XLSX document][xlsx]{:target="_blank"} and output its 
full filename. Hence generating a [PDF][pdf]{:target="_blank"} export is 
trivial with:

{% highlight console %}
$ checkbox-cli tp-export com.canonical.certification::client-cert-18-04 | xargs -d '\n' libreoffice --headless --invisible --convert-to pdf
{% endhighlight %}

# Tips and tricks

## Change the title of an existing session

Forgot to run checkbox with `--title` or simply unhappy with the title given to 
your session? You can easily change it by editing the `submission.json` inside
the archive. Let's use `vim submission.tar.xz` to browse the tarball content and
edit the JSON report:

![image tooltip here](/assets/images/title.svg)

## Add launchpad bug links in comments

Checkbox does not offer a convenient method to link a bug to a failed test 
case. But with the same method used above, you can edit the JSON report and add
some HTML markup. They will be used as is while re-exporting the session using 
`checkbox-cli merge-submissions`:

![image tooltip here](/assets/images/bug_link.svg)

Open the new tarball and check the HTML report, it now contains our bug link:

![bug_link](/assets/images/bug_link.png)

[xlsx]: {{site.url }}/assets/files/18.04_Client_Certification_Tests.xlsx
[pdf]:  {{site.url }}/assets/files/18.04_Client_Certification_Tests.pdf
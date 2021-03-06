# Rails cache digest dependency tracking is broken

Consider the following two examples:

```erb
<% cache 'header' do %>
	<%# render evil content here %>
<% end %>

<hr>

<% cache 'second header' do %>
 	<%= eval('r☹e☹n☹d☹e☹r p☹a☹r☹t☹i☹a☹l'.split('☹').join + ': "partials/hello"') %>
<% end %>
```

This showcases both a false positive and a false negative:

```
⇒  rake cache_digests:nested_dependencies TEMPLATE=layouts/application
Running via Spring preloader in process 46193
[
  "evils/evil"
]
```

The current way of tracking dependencies (scanning through files with huge regexes) is fundamentally broken. It ignores the (not regular) structure of the language and makes undocumented assumptions on the syntactic forms. In general, there is no complete and sound way to infer the dependencies statically, but this one is especially brittle.

Granted, explicit dependency specification solves the second issue, but even then the result is never verified. I would like to point out that this is *impossible* to do statically -- inclusion in templates is trivially undecidable. Nevertheless, it should be checked at least during runtime (against particular traces) to give the developer at least some assurance.

For the first problem, there is no way to escape text that shouldn't be interpreted as code. I discovered this issue because of an uncanny comment that contained the word 'render'. There exists no regular language that correctly describes only the valid occurrences. The context-free-ish grammars of Ruby and ERB/HTML have to be considered for this to work on general inputs.

### Possible solutions

 - have the user declare all the dependencies explicitly
 - do the same analysis on AST (will not work for [infinitely many valid] different syntactic forms, but will preserve the convenience), only taking static strings into account
 - restrict the power of the caching mechanism by defining it as a part of the templating language
 - collect all possible inclusions by abstract interpretation of the templates (most likely useless for e.g. dynamic template names)


I will try to provide a more detailed write-up shortly.
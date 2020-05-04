# Model-View-Template Architecture

## MVC Pattern
I first learned about the [MVC design pattern](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) when learning [ruby on rails](https://rubyonrails.org/), which is another great web development framework. I shifted to python when I realized I could write code across our entire stack (ETL, web development, API, data analysis, etc) in a consistent language.

There are plenty of resources on MVC across the web, but this diagram is helpful for understand how a request for a web page travels through the different interfaces and results in rendering data-backed dynamic content to end users. 
 
[[images/mvc_pattern.png]]

## MTV?

It's the same but different! It's a common enough point of confusion that [Django covers it in their FAQ](https://docs.djangoproject.com/en/2.2/faq/general/#django-appears-to-be-a-mvc-framework-but-you-call-the-controller-the-view-and-the-view-the-template-how-come-you-don-t-use-the-standard-names). 

My diagram doesn't exactly match their explanation, but it's how I best conceptualize the similarities.


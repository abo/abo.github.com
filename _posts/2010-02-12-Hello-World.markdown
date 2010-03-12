---
layout: post
title: Hello World
---

## HelloWorld

* Hello World!

Hello World!

{% highlight ruby %}
module XSLTRender
  def apply_xslt(template_file, xml)
    # use xsltproc to perform the transformation
  end

  def instance_hash
    hash = instance_variables.inject({}) do |vars, var|
      key = var[1..-1]
      vars[key] = instance_variable_get(var)
      vars
    end

    hash['flash'] = flash
    hash
  end

  def render_through_xslt(options = {})
    template = find_template(options)
    page_xml = instance_hash.to_xml
    html = apply_xslt(template, page_xml)
    render :text => html
  end
end
{% endhighlight %}

{% highlight csharp %}
	public static void Main()
	{
		Console.WriteLine("Hello World!");
	}
{% endhighlight %}

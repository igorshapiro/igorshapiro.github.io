---
title: First post
layout: post
---

Code highlighting test
{% highlight ruby %}
class Cls
  def show
    @widget = Widget(params[:id])
    respond_to do |format|
      format.html # show.html.erb
      format.json { render json: @widget }
    end
  end
end
{% endhighlight %}

---
layout: books
icon: fas fa-book
order: 4
---
<p>As a software engineer, you probably have read many books. They can prove a very useful resource, as many contain valuable information on various technology topics, be it specific technologies or more general approaches on a specific area.</p>

<p>This is a list of my favorite books, if you have any book to recommend please share your thoughts below, I will definitely give it a read and post it here! Purpose of this list is not to showcase how many books I have read, or not, but to spread word on good books which helped me learn new things and expand my horizons!</p>
<h2>Top picks</h2>

<div class="image-holder">
{% for book in site.data.books %}
	<div class="image">
    <figure>
      <a href="{{book.srcUrl}}">
        <img 
          src="{{book.imgUrl}}"
          alt="{{book.caption}}">
      </a>
      <figcaption>{{book.caption}}</figcaption>
    </figure>
  </div>
{% endfor %}
</div>

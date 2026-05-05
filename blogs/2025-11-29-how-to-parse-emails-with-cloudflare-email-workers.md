---
title: "How to parse emails with Cloudflare Email Workers?"
url: "https://blog.emailengine.app/how-to-parse-emails-with-cloudflare-email-workers/"
date: "Sat, 29 Nov 2025 10:11:00 GMT"
author: "Andris Reinman"
feed_url: "https://blog.emailengine.app/rss/"
---
<blockquote>This blog post is about general email processing with Cloudflare Email Workers and is not specific to EmailEngine. If you want to process incoming emails with EmailEngine instead, see the other posts <a href="https://blog.emailengine.app/tag/email-engine/">here</a>.</blockquote><img alt="How to parse emails with Cloudflare Email Workers?" src="https://blog.emailengine.app/content/images/2024/02/CF_logo_stacked_blktype.jpg" /><p>Cloudflare <a href="https://developers.cloudflare.com/email-routing/email-workers/?ref=blog.emailengine.app">Email Workers</a> are a nifty way to process incoming emails. The built-in API of Cloudflare Workers allows you to route these emails, and it also provides some email metadata information. For example, you can reject an incoming email with a bounce response, you can forward it, or you can generate and send a new email. Your worker is also provided with the SMTP envelope information, like the envelope-<em>from </em>and envelope-<em>to</em> addresses used for routing.</p><pre><code class="language-javascript">export default {
  async email(message, env, ctx) {
    message.setReject(&quot;I don&apos;t like your email :(&quot;);
  }
}
</code></pre><p>All emails to such an email route will bounce with the provided message.</p><figure class="kg-card kg-image-card"><img alt="How to parse emails with Cloudflare Email Workers?" class="kg-image" height="1226" src="https://blog.emailengine.app/content/images/2024/02/Screenshot-2024-02-22-at-14.22.02.png" width="1498" /></figure><p>But what about the content? Email routing information is mainly relevant for routing but not so much for processing. For example, it is probably not useful at all to detect the sender address as something like <em>bounce-mc.us20_123456789.17649072-1234567899@mail30.atl18.mcdlv.net</em>. It only tells us that this email was sent through a Mailchimp mailing list, but it does not reveal the actual sender. And what about the subject of the email or HTML body?</p><p>It turns out Cloudflare provides <em>some</em> information about the email contents. The message object includes a <code>headers</code> object which you can use to read email headers. It makes it really easy to read stuff like email subject line:</p><pre><code class="language-javascript">let subject = message.headers.get(&apos;subject&apos;);
</code></pre><p>The <code>headers.get()</code> method is good for reading single-line values like the subject line but kind of falls through when processing headers that might have multiple values like the <code>To:</code> or<code> Cc:</code> address lines. Additionally, there is no information at all about the text contents of the email or attachments.</p><p>Luckily, the message object includes an additional property called <code>raw</code>, which is a readable stream. From that stream, we can read the source code of the email, which in itself, yet again, is not very useful, but we can parse it to get any information we need about the email. Email parsing is quite complex and difficult, but luckily, there is a solution: the <a href="https://www.npmjs.com/package/postal-mime?ref=blog.emailengine.app">postal-mime</a> package.</p><p>All you need to do is to install <a href="https://www.npmjs.com/package/postal-mime?ref=blog.emailengine.app">postal-mime</a> dependency from NPM.</p><pre><code>npm install postal-mime
</code></pre><p>And import it into your worker code.</p><pre><code class="language-javascript">import PostalMime from &apos;postal-mime&apos;;
</code></pre><p>This allows you to easily parse incoming emails.</p><pre><code class="language-javascript">const email = await PostalMime.parse(message.raw);
</code></pre><p>The resulting parsed <code>email</code> object includes a bunch of stuff like the subject line (<code>email.subject</code>) or the HTML content of the email (<code>email.html</code>). You can find the full list of available properties from the <a href="https://www.npmjs.com/package/postal-mime?ref=blog.emailengine.app#parserparse">docs</a>.</p><h3 id="attachment-support">Attachment support.</h3><p>PostalMime parses all attachments into <code>ArrayBuffer</code> objects. If you want to process the contents of an attachment as a regular string (which makes sense for textual attachments but not for binary files like images) or as a base64 encoded string, you can use the <code>attachmentEncoding</code> configuration option.</p><pre><code class="language-javascript">const email = await PostalMime.parse(message.raw, {
    // Using &quot;utf8&quot; makes only sense with text files like .txt or .md
    // For binary files likes images or PDF, use &quot;base64&quot; instead
    attachmentEncoding: &apos;utf8&apos;
});
console.log(email.attachments[0].content);</code></pre><p>If you need to process binary attachments as strings, converting the ArrayBuffer value into a base64 encoded string is probably best. Set the <code>attachmentEncoding</code> configuration option to &quot;base64,&quot; and that&apos;s it; you can now process binary attachments safely as strings.</p><h3 id="full-example">Full example</h3><p>The following Email Worker parses an incoming email and logs some information about the parsed email to the worker&apos;s log output.</p><pre><code class="language-javascript">import PostalMime from &apos;postal-mime&apos;;

export default {
  async email(message, env, ctx) {
    const email = await PostalMime.parse(message.raw, {
        attachmentEncoding: &apos;base64&apos;
    });

    console.log(&apos;Subject&apos;, email.subject);
    console.log(&apos;HTML&apos;, email.html);

    email.attachments.forEach((attachment) =&gt; {
      console.log(&apos;Attachment&apos;, attachment.filename, attachment.content);
    });
  },
};
</code></pre>

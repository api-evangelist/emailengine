---
title: "Mailbox locking in ImapFlow"
url: "https://blog.emailengine.app/mailbox-locking-in-imapflow/"
date: "Mon, 19 Jul 2021 07:57:17 GMT"
author: "Andris Reinman"
feed_url: "https://blog.emailengine.app/rss/"
---
<blockquote><a href="https://imapflow.com/?ref=blog.emailengine.app">ImapFlow</a> is an IMAP access module for Node.js. It is used by IMAP API under the hood to make connections to IMAP servers and to run commands.</blockquote><p>ImapFlow library allows opening folders in an IMAP account via two different methods, which are <a href="https://imapflow.com/module-imapflow-ImapFlow.html?ref=blog.emailengine.app#mailboxOpen"><em>mailboxOpen(path)</em></a> and <a href="https://imapflow.com/module-imapflow-ImapFlow.html?ref=blog.emailengine.app#getMailboxLock"><em>getMailboxLock(path)</em></a>. What is the actual difference and why would you need something like that?</p><p>Think of the following. More or less at the same time, maybe due to user actions, our application tries to list all unseen emails in Inbox and delete all emails in Trash. These are the functions we run at the same time using the same IMAP connection:</p><pre><code class="language-javascript">async function listUnseen(path){
    await imap.openBox(path);
    let list = await imap.search(&apos;1:*&apos;, &apos;UNSEEN&apos;);
    return list;
}

async function deleteAll(path){
    await imap.openBox(path);
    await imap.addFlags(&apos;1:*&apos;, &apos;\\Deleted&apos;);
    await imap.expunge();
}
</code></pre><p>IMAP connection does not run commands in parallel, you always have to wait until the previous command finishes until you can run the next one. So it is easy to see that we are running into conflicts if we queue a bunch of commands at the same time and then try to run these:</p><!--kg-card-begin: html--><table>
<thead>
<tr>
<th>List all unseen</th>
<th>Delete all from Trash</th>
</tr>
</thead>
<tbody>
<tr>
<td><em>idle</em></td>
<td><code>SELECT Trash</code></td>
</tr>
<tr>
<td><em>waiting</em></td>
<td><code>OK selected Trash</code></td>
</tr>
<tr>
<td><code>SELECT INBOX</code></td>
<td><em>waiting</em></td>
</tr>
<tr>
<td><code>OK selected INBOX</code></td>
<td><em>waiting</em></td>
</tr>
<tr>
<td><em>waiting</em></td>
<td><code>STORE 1:* (\Deleted)</code></td>
</tr>
<tr>
<td><em>waiting</em></td>
<td><code>OK store completed</code></td>
</tr>
<tr>
<td><code>SEARCH UNSEEN</code></td>
<td><em>waiting</em></td>
</tr>
<tr>
<td><code>* SEARCH 1,2,&#x2026;</code></td>
<td><em>waiting</em></td>
</tr>
<tr>
<td><code>OK search completed</code></td>
<td><em>waiting</em></td>
</tr>
<tr>
<td><em>idle</em></td>
<td><code>EXPUNGE</code></td>
</tr>
<tr>
<td><em>idle</em></td>
<td><code>* 1 EXPUNGE&#x2026;</code></td>
</tr>
<tr>
<td><em>idle</em></td>
<td><code>OK expunge completed</code></td>
</tr>
</tbody>
</table><!--kg-card-end: html--><p>So what happened here was that we actually deleted all the emails in the INBOX and not from the Trash. Not exactly what we wanted, isn&apos;t it?</p><p>ImapFlow tries to address this issue by using mailbox locking. You lock the mailbox, run your commands and release the lock. All other actions must wait until the lock is released. So it is kind of like a soft transaction, except that it does not roll back if exceptions occur.</p><p>After small modifications our code now looks like this:</p><pre><code class="language-javascript">async function listUnseen(path){
    let lock = await client.getMailboxLock(path);
    try {
        return await client.await client.search({unseen: true});
    } finally {
        lock.release();
    }
}

async function deleteAll(path){
    let lock = await client.getMailboxLock(path);
    try {
        await client.messageDelete(&apos;1:*&apos;);
    } finally {
        lock.release();
    }
}
</code></pre><p>This time commands can not be queued at the same time and the resulting action seems different:</p><!--kg-card-begin: html--><table>
<thead>
<tr>
<th>List all unseen</th>
<th>Delete all from Trash</th>
</tr>
</thead>
<tbody>
<tr>
<td><em>idle</em></td>
<td><code>SELECT Trash</code></td>
</tr>
<tr>
<td><em>waiting</em></td>
<td><code>OK selected Trash</code></td>
</tr>
<tr>
<td><em>waiting</em></td>
<td><code>STORE 1:* (\Deleted)</code></td>
</tr>
<tr>
<td><em>waiting</em></td>
<td><code>OK store completed</code></td>
</tr>
<tr>
<td><em>waiting</em></td>
<td><code>EXPUNGE</code></td>
</tr>
<tr>
<td><em>waiting</em></td>
<td><code>* 1 EXPUNGE&#x2026;</code></td>
</tr>
<tr>
<td><em>waiting</em></td>
<td><code>OK expunge completed</code></td>
</tr>
<tr>
<td><code>SELECT INBOX</code></td>
<td><em>idle</em></td>
</tr>
<tr>
<td><code>OK selected INBOX</code></td>
<td><em>idle</em></td>
</tr>
<tr>
<td><code>SEARCH UNSEEN</code></td>
<td><em>idle</em></td>
</tr>
<tr>
<td><code>* SEARCH 1,2,&#x2026;</code></td>
<td><em>idle</em></td>
</tr>
<tr>
<td><code>OK search completed</code></td>
<td><em>idle</em></td>
</tr>
</tbody>
</table><!--kg-card-end: html--><p>So what happens is that operations become slightly slower as they need to wait until all other actions are finished but there aren&apos;t any more conflicts and we do not end up deleting messages from the wrong folder.</p>

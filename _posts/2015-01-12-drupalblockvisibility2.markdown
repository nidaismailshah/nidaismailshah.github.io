---
layout: post
title:  "Managing Block Visibility in Drupal - 2."
date:   2015-01-12 14:00:00
categories: drupal
tag: 1
---

Create a custom table and define various fields in it using ```hook_schema()```.


This blog post is in continuation with the previous <a href="/drupal/2015/01/05/drupalblockvisibility1/">blog post</a>.


To manage blocks' visibility we need to have parameters that would determine whether a block would be visible or not.
The parameters that we need to set when we create or edit a block are:
<ul>
<li><em>Pages</em> : to restrict blocks' visibility on certain pages.</li>
<li><em>Content Types</em> : to restrict blocks' visibility on certain content types.</li>
<li><em>Roles</em> : to restrict blocks' visibility for certain user roles.</li>
<li><em>Users</em> : to allow per user customizations for blocks.</li>
</ul>
These parameters are available out of the box in any drupal site. To create custom parameters refer to this <a href="/drupal/2015/01/05/drupalblockvisibility1/">blog post</a>.
We need to save these parameters/configurations into the database in a custom table. To create a custom table we can use ```hook_schema()```. ```hook_schema()``` takes care of the table creation and deletion at the time of module installation and un-install respectively. In the following code, I've created a custom table named ```block_date``` inside the ```hook_schema()``` in the my_module.install file. Read more about hook_schema() <a href="https://api.drupal.org/api/function/hook_schema">here</a> and .install files <a href="https://www.drupal.org/node/876250">here</a>.
{% highlight ruby %}
/**
 * Implements hook_schema().
 */
function my_module_schema() {
  $schema['block_date'] = array(
    'description' => 'Sets up display criteria for blocks based on post date',
    'fields' => array(
      'module' => array(
        'type' => 'varchar',
        'length' => 64,
        'not null' => TRUE,
        'description' => "The block's origin module, from {block}.module.",
      ),
      'delta' => array(
        'type' => 'varchar',
        'length' => 32,
        'not null' => TRUE,
        'description' => "The block's unique delta within module, from {block}.delta.",
      ),
      'res_date' => array(
        'type' => 'varchar',
        'length' => 32,
        'not null' => TRUE,
        'description' => "The date this block's visibility should be restricted according to.",
      ),
      'start' => array(
        'type' => 'varchar',
        'length' => 32,
        'description' => "Visibility before or after",
      ),
    ),
    'primary key' => array('module', 'delta', 'start'),
  );
  return $schema;
}
{% endhighlight %}
Here ```'fields'``` is an array of fields. So the table ```block_date``` would be created when we install our module and it would have the above defined fields.
So we have our custom table created and data would be queried into it when a block is edited or a new block is created. The code for the database query would look like this:
{% highlight ruby %}
$query = db_insert('block_date')->fields(array('res_date', 'start','module', 'delta'));
    $query->values(array(
      'res_date' => $form_state['values']['res_date'],
      'start' => $form_state['values']['start'],
      'module' => $form_state['values']['module'],
      'delta' => $form_state['values']['delta'],
    ));
  $query->execute();
}
{% endhighlight %}
This is to be put inside the submit handler of the form we altered using ```hook_form_FORM_ID_alter()``` in the previous blog post so that the query runs when the form is submitted after some validations, of course.

Duh! We have a problem there, Know what?

If we edit a block whose visibility parameters are already set, and we happen to change the parameters and submit the form again, we would be querying into the database two different parameters of visibility for the same block. And that, sure as hell, is a problem.

To avoid this we would need to delete the previously set parameters for that block from the database. To do that we use the following query before inserting new data into the database in the submit handler of the ```block_admin_configure``` form.
{% highlight ruby %}
// #delete already set date configurations
  db_delete('block_date')
    ->condition('module', $form_state['values']['module'])
    ->condition('delta', $form_state['values']['delta'])
    ->execute();
{% endhighlight %}

So now we have a block created or edited and the parameters collected from it and stored into a custom table in our database. Now we need to use these parameters to determine blocks' visibility. I would be writing another blog post to explain how to use these parameters to manage a block's visibility.

Stay tuned!
---
layout: post
title:  "Managing Block Visibility in Drupal."
date:   2015-01-05 14:35:05
categories: drupal
---

All the  customizations in Drupal are done using hooks.

<h1>Hooks</h1>
Hooks are Drupal's way of giving developers a chance (by calling hooks in their custom modules) to customize Drupal Core functionality and presentation. Read more about hooks <a href ="https://3cwebservices.com/drupal/introduction-drupal-hooks" target = "-blank">here</a>. 

To manage a block's visibility, Drupal has already set a few options. These include:
<ul>
 <li>Visibility according to specific pages.</li>
 <li>Visibility for different content types.</li>
 <li>Visibility for different User roles.</li>
 <li>Visibility for different Users.</li>
 </ul>
We can find these options in the add-block form when we add a new block or edit an existing block. <img src = "/images/drupal/1.png">Now If we research on how Drupal really does it and where this functionality is coming from, We'll end up at node module in Drupal Core for content type specific visibility and block module in Drupal Core for other options. The code would look somewhat like this:
{% highlight ruby %}
 /**
 * Implements hook_form_FORMID_alter().
 *
 * Adds node-type specific visibility options to block configuration form.
 *
 * @see block_admin_configure()
 */
function node_form_block_admin_configure_alter(&$form, &$form_state) {
  $default_type_options = db_query("SELECT type FROM {block_node_type} WHERE module = :module AND delta = :delta", array(
    ':module' => $form['module']['#value'],
    ':delta' => $form['delta']['#value'],
  ))->fetchCol();
  $form['visibility']['node_type'] = array(
    '#type' => 'fieldset',
    '#title' => t('Content types'),
    '#collapsible' => TRUE,
    '#collapsed' => TRUE,
    '#group' => 'visibility',
    '#weight' => 5,
  );
  $form['visibility']['node_type']['types'] = array(
    '#type' => 'checkboxes',
    '#title' => t('Show block for specific content types'),
    '#default_value' => $default_type_options,
    '#options' => node_type_get_names(),
    '#description' => t('Show this block only on pages that display content of the given type(s). If you select no types, there will be no type-specific limitation.'),
  );
  $form['#submit'][] = 'node_form_block_admin_configure_submit';
}
{% endhighlight %}
hook_form_FORM_ID_alter() has been implemented in the above code which sets the various visibility settings for the add-block-form. The vertical tabs are represented by form elements which are fieldsets like: 
{% highlight ruby %}
 $form['visibility']['node-type']
{% endhighlight %}
To create our custom, our code should implement the same hook_form_FORM_ID_alter() and define a fieldset as:
{% highlight ruby %}
$form['visibility']['our custom tab'] = array(
// configurations
);
{% endhighlight %}
We can also keep various elements inside our custom tab and use them to obtain values from users such as node types, date etc and customize the block visibility based on these values.
{% highlight ruby %}
$form['visibility']['our custom tab']['our element'] = array(
 '#type' => 'textfield' // say
 // other configurations.
);
{% endhighlight %}

Let's say we had to restrict the visibility of blocks before or after a certain date. To do this job we would need to ask users for the date. we can do that by simply defining a custom tab as above and asking users to enter the date in a text field and then use the date value to restrict the blocks' visibility. Just like this: <img src = "/images/drupal/4.png">
The code for this would look somewhat like this:
{% highlight ruby %}
/**
 * Implements hook_form_FORMID_alter().
 *
 * Adds node-type specific visibility options to block configuration form.
 *
 * @see block_admin_configure()
 */
 function my_module_form_block_admin_configure_alter(&$form, &$form_state) {
  $prev_set_date_config = db_query("SELECT res_date, start FROM {block_date} WHERE module = :module AND 
    delta = :delta", array(
    ':module' => $form['module']['#value'],
    ':delta' => $form['delta']['#value'],
    ))->fetch(PDO::FETCH_BOTH);
  // The Post Date vertical tab.
  $form['visibility']['post_date'] = array(
    '#type' => 'fieldset',
    '#title' => t('Post Date'),
    '#collapsible' => TRUE,
    '#collapsed' => TRUE,
    '#group' => 'visibility',
    '#weight' => 30,
  );

  // The radio button in the post date vertical tab.
  $form['visibility']['post_date']['start'] = array(
    '#type' => 'radios',
    '#title' => t('Show block for nodes created'),
    '#default_value' => $prev_set_date_config[1],
    '#options' => array(
    	'after' => t('On or after below date'),
    	'before' => t('Prior to below date'),
    	),
  );

  // The date field in the post date vertical tab.
  $form['visibility']['post_date']['res_date'] = array(
  	'#type' => 'textfield',
  	'#title' => t('Date'),
  	'#default_value' => $prev_set_date_config[0],
  	'#description' => t('Date in MM/DD/YYYY format'),
  	'#size' => 10,
  	'#maxlength' => 10,
  	'#weight' => 10,
  	);
}
{% endhighlight %}

I'll soon be writing another blogpost to explain how to restrict a block's visibility on a certain page. Stay tuned!
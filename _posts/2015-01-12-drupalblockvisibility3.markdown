---
layout: post
title:  "Managing Block Visibility in Drupal - 3."
date:   2015-01-12 18:00:00
categories: drupal
tag: 1
---
Allowing or restricting the blocks' visibility using ```hook_block_list_alter($blocks)```.

This hook is implemented by all modules that need to alter the list of blocks to b rendered. The block list contains the block definitions and not the rendered blocks. So we can very well add or remove a block or even alter its contents as per our needs. Read more about this hook <a href="https://api.drupal.org/api/drupal/modules!block!block.api.php/function/hook_block_list_alter/7"> here</a>.This hook takes as input an array of blocks ```$blocks[]``` with block ids of blocks as keys.

Now to determine the block's visibility we have to keep or remove a particular block from this block list or this array of $blocks. If we want a block to be visible, we keep it in the list otherwise we remove it off the list and we do that by deleting the element in the $blocks array corresponding to that block. We can simply use php ```unset()``` function to perform the job like this: 
{% highlight ruby %}
unset($blocks[$key]);
{% endhighlight %}
where $key holds the block id of the block whose visibility is to be restricted.
The code would look somewhat like this:
{% highlight ruby %}
function my_module_block_list_alter(&$blocks) {
	//# traverse through the $blocks array
	foreach ($blocks as $key => $block) {
		//# check for the visibility
    	if (!visibility_check($some_parameter)){
    			//# delete the block off the list/array
               unset($blocks[$key]);
               continue;
        }
    }
}
{% endhighlight %}

Now to restrict a blocks visibility on the basis of date of creation of a node we would need to compare the two dates, one from the custom table ```block_date``` and one from the node object ```$node->created```. To do this job, we would need to query the custom table ```block_date``` in our database, retrieve the restriction date ```$res_date``` and then use some function to compare this date with the ```node->created``` date obtained from the node object. The query that would retrieve the date of restriction from ```block_date``` table would look somewhat like this:
{% highlight ruby %}
 // #array to store block visibility date
  $block_date_start = array();
  $result = db_query('SELECT module, delta, start, res_date FROM {block_date}');
  foreach ($result as $record){
    $block_date_start[$record->module][$record->delta]['start'] = $record->start;
    $block_date_start[$record->module][$record->delta]['res_date'] = $record->res_date;
  }
{% endhighlight %}
and this shows how to retrieve the unix timestamp date of node creation from node object and format it as per our use:
{% highlight ruby %}
$node_created = format_date($node->created, 'custom', 'm/d/Y');
{% endhighlight %}
So now we have the two dates available to us and we need to compare the two dates to determine whether the block would be visible on a particular node or not. The final code in the ```hook_block_list_alter``` would look like this:
{% highlight ruby %}
function my_module_block_list_alter(&$blocks) {
  global $theme_key;

  //#array to store block visibility date
  $block_date_start = array();
  $result = db_query('SELECT module, delta, start, res_date FROM {block_date}');
  foreach ($result as $record){
    $block_date_start[$record->module][$record->delta]['start'] = $record->start;
    $block_date_start[$record->module][$record->delta]['res_date'] = $record->res_date;
  }

  $node = menu_get_object();

  foreach ($blocks as $key => $block) {
    if(isset($block_date_start[$block->module][$block->delta])) {
      if (!empty($node)) {
          $node_created = format_date($node->created, 'custom', 'm/d/Y');
          $start = $block_date_start[$block->module][$block->delta]['start'];
          $res_date = $block_date_start[$block->module][$block->delta]['res_date'];

          if (!visibility_check($node_created, $start, $res_date)){
               unset($blocks[$key]);
               continue;
          }
      }
    }
  }
}
{% endhighlight %}
Including this hook in our ```my_module.module``` file would certainly add the functionality to our drupal site where in users/site administrators would be able to decide whether a block would be visible on a node before or after a certain date or not. The function ```visibility_check()``` is a simple php date comparison function and has nothing to do with drupal blocks. This is something we can manage to code easily.

However there are some more things that need to be taken care of before we are finally done with adding this funcionality to our drupal site. Those things would be covered in another blog post.

Stay tuned!
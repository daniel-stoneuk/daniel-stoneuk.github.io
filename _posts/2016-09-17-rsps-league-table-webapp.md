---
published: true
layout: post
date: '2016-09-17 13:51 +0100'
title: RSPS - League Table WebApp
---
## Throwbacks.

Around Christmas last year, my school form and I decided to host a league based competition around the Android game [Pocket Soccer][pocketsoccer] - to reminisce our year 7 days (3 years earlier).  I, being the true tech geek I am decided to use my, at the time, newfound PHP knowledge to develop a pretty complex website to keep track of the games. I worked on it during lunch break and in the evening for about a week, until it was finally finished. I am impressed by the amount of work I put into it back then.

I worked with the [Materialize][] framework to make it look pretty, but the bulk of the logic was in the backend - in PHP and SQL. I incorporated a proper login system with randomly generated keys which would tie a person to a user account. You can check it out here: [rsps.daniel-stone.uk][rsps].

![rsps1.png]({{site.baseurl}}/assets/rsps1.png)

If you're not logged in (you won't be) all you will see is the main homepage. Here is a little snippet of the PHP code behind generating a tab. I grab the id's from the users table where the group is the currently iterating group and order them by alphabetical order or points depending on the users choice.

I then fill in a table row using `echo` and the needed html code.

```php
<?php

...

$query = "SELECT `id` FROM `users` WHERE `1stgroup`='".$group."' ".(isset($_GET['order']) ? "ORDER BY `fullname` ASC" : "ORDER BY `won` DESC, `goaldifference` DESC;");

$result = mysqli_query($link, $query);
$idArray = array();
while($row = mysqli_fetch_array($result)){
	$idArray[] = $row['id'];
};
$position = "1";
foreach ($idArray as $id){
	$query = "SELECT * FROM users WHERE id='".$id."' LIMIT 1";
	$result = mysqli_query($link, $query);
	$row = mysqli_fetch_array($result);
	$firstLetter = substr($row['fullname'], 0, 1);
	echo "<tr><td style='font-weight: 300;' class='table-center'>".( isset($_GET['order']) ? $firstLetter : $position)."</td><td>".htmlspecialchars($row['nickname']);
	if (htmlspecialchars($row['paid']) == 0){ echo "<span class='my-badge red accent-3 material-icons'>money_off</span>";};
	if (htmlspecialchars($row['paid']) == 1){ echo "<span class='my-badge green accent-4 material-icons'>attach_money</span>";}
echo "
	</td><td class='table-center'>". htmlspecialchars($row['played'])."
	</td><td class='table-center'>".htmlspecialchars($row['won'])."
	</td><td class='table-center'>".htmlspecialchars($row['lost'])."
	</td><td class='table-center'>".htmlspecialchars($row['goaldifference'])."
	</tr>";
	$position += 1;
};
echo "</tbody></table></div>";
?>
};
```

If anyone want's to have my code, I would be more than happy to open source it - as there's no point of it just sitting there. Not too sure whether it is worth it yet as nobody needs it. 

Maybe someday I will reuse it for a different league based competition. I hope this has been somewhat interesting, and useful for anyone else thinking of undergoing a similar project.

Oh, and I optimised it for mobile.

![rsps.gif]({{site.baseurl}}/assets/rspsgif.gif){:.screenshot}


Catch you later,  
Dan

[pocketsoccer]: 	https://play.google.com/store/apps/details?id=com.rastergrid.game.pocketsoccer&hl=en
[Materialize]: 	http://materializecss.com/
[rsps]: 	http://rsps.daniel-stone.uk

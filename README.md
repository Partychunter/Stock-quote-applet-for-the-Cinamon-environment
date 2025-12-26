# Stock-quote-applet-for-the-Cinamon-environment
Stock exchange rates applet for the Cinnamon environment - displays exchange rates and several selected indices relevant to a Polish resident on the Panel and in an expandable list
Installation and use
Version 1: Predefined link text (email-obfuscator-text-link.js)

HTML: Place an <a> element with a unique id, e.g. contactLink, and any text.

<html>
  <a id="kontaktLink" href="#">Contact us</a>
</html>
<script LANGUAGE="JavaScript">
  var user = "yourname"; 
  var site = "yourdomin"; 
  var emailLink = "mailto:" + user + "@" + site;
  var kontaktLink = document.getElementById("kontaktLink");
  kontaktLink.href = emailLink;
</script>


### Version 2: Displaying email address (email-obfuscator-display-email.js)


<script LANGUAGE="JavaScript">
  var user = "yourname"; 
  var site = "yourdomin"; 

  document.write('<a href=\"mailto:' + user + '@' + site + '\">');
  document.write(user + '@' + site + '</a>');
</script>

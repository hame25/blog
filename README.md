# 103 Early Hints

For a dynamic web application TTFB (Time To First Byte) can take time. For each request the server recieves in order to respond correctly it could be performing a number of things that can add up; multiple api requests (some in a serial nature), computation of data, constructing a html response etc. All this takes time and whilst this is happening the browser is just… waiting!

#HCI 574 RSS Reader application


import xml.etree.ElementTree as etree # elementtree XML parser module, see http://docs.python.org/library/xml.etree.elementtree.html
import urllib2  # urllib2 is more efficient than urllib ("1")
import cgi
import webapp2


#Google App Engine to be able to deploy the app
USE_GOOGLE_APPENGINE = True
if USE_GOOGLE_APPENGINE == False:
    import sys, pprint, os, glob
    GA_lib=r"C:\Program Files (x86)\Google\google_appengine\lib"
    for d in glob.glob(GA_lib + os.sep + "*"):
        sys.path.append(d)
    pprint.pprint(sys.path)
    
#writing to HTML
class MainPage(webapp2.RequestHandler):
    def get(self):
        self.response.out.write(
        """
      <!DOCTYPE html>  
        <html>
             <body>
             Google News Summary for:<br>
               <form action="/result" method="post">
                     <input type="text" name="search_term"
                  <input type="submit" value="search">
               </form>
             </body>
        </html> 
        """
        )
import sgmllib
class MyParser(sgmllib.SGMLParser):  # subclass new parser from SGMLParser

    def __init__(self, verbose=0): # call superclass' init()
        sgmllib.SGMLParser.__init__(self, verbose)
        self.hyperlinks = []  # list to later store hyperlinks found in html_string

    def consume(self, html_string): # Parse a string with HTML text
        self.feed(html_string) # feed() will call start_a() whenever a <a> tag is detected
        self.close()

    def start_a(self, attributes):  # called when a  <a> tag is found
        #Process the <a> tag and look for href attribute
        for name, value in attributes: # list of [
            if name == "href": 
                self.hyperlinks.append(value) # found a href in <a> tag

class ShowResult(webapp2.RequestHandler):
    def post(self):
        
        #What we are searching for
        search_term = raw_input("Topic:") #MAY CAUSE ISSUES IF THE SELF RESPONSE OUT WRITE FUNCTION IS CLASHING
        search_term = "%22" + search_term + "%22"
        search_term = search_term.replace(" ", "%20")
        rss_feed_URL = 'http://news.google.comnews?q={:s}&output=rss'.format(search_term)
        
        
        #getting the xml file from the server & saving locally
       feed_socket = urllib2.ulopen(rss_feed_URL)
        xml_content = feed_socket.read()
        feed_socket.close() #NOT SURE IF THIS IS ACTUALLY NECESSARY
        with open("RSS feed for" + search_term + ".xml", "w+") as xml_file:
            xml_file.write(xml_content)
        
        #Parse the feed xml "text" with element tree
        feed_socket = urllib2.urlopen(rss_feed_URL)
        tree = etree.parse(feed_socket)
        feed_socket.close()
        
        #grabbing the tags for title, date & link
        channel = tree.find("channel")
        items_list = channel.findall("item")
        self.response.write('<html><body>') 
        for i in items_list:
            title_tag = i.find("title")
            pubDate_tag = i.find("pubDate")
            link_tag = i.find("link")
            
            time = pubDate_tag.text.split()[4]
            title = title_tag.text.replace('\n', ' ')
            linkURL = link_tag.text
            
            anchor_tag = "<a href={:s}>{:s}</a>".format(linkURL, title)
            self.response.write(anchor_tag + "<br>" + time)
            print i
        self.response.write('</body></html>')


#instantiate a Web Server Gateway Interface (WSGI) application
app = webapp2.WSGIApplication([('/', MainPage),
                               ('/result', ShowResult)],
                              debug = True)




#Running if not able to run on Google App Engine
if USE_GOOGLE_APPENGINE==False:
    from paste import httpserver
    print "running local httpserver(copy/paste this URL into your browser),",
    httpserver.serve(app, host='127.0.0.1', port= '8080')
    
        

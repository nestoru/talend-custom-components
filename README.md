[![](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/donate/?hosted_button_id=58F9TDDRBND4L)

talend-custom-components
========================

This README is an attempt to write a Talend Custom Component creation tutorial. 

We will build a custom component which implements a CSV Filter (which provides schemaless filtering on CSV files). 

For the complete use case you can read  http://thinkinginsoftware.blogspot.com/2012/10/the-data-team-was-in-need-of.html where I show how to build a CSV Filter use a tJavaFlex component.

With the hope of bigger Talend Open Source contributions I have put this project available to the community here at github. There are a lot of open requests and features even for existing talend components so you could fork this project and show Talend community your fixes or additions to the current code base. I believe Talend team should use github just as Oracle has done with Java and VMWare has done with Spring Framework.

Deploying the component
=======================
1. cd /tmp
1. git clone https://github.com/nestoru/talend-custom-components.git
1. cp -R /tmp/talend-custom-components/tFileInputCSVFilter $TALEND_HOME/plugins/org.talend.designer.components.localprovider_*/components/
1. restart talend or press ctrl+shift+F3
 
Alternatively you could use any of the components without using git at all. Just download the project using https://github.com/nestoru/talend-custom-components/downloads and then deploy the specific component directory following the steps at http://thinkinginsoftware.blogspot.com/2012/10/install-custom-talend-component.html )

Getting started
===============
1. Create a directory for your custom components, for example /opt/talend-custom-components
1. Go to "Preferences|Talend Component Designer" use the directory path for "Component Project"
1. In "Preferences|Talend/Components" use the directory path for "User component folder". Click OK
1. Select "Window|Perspective|Component Designer", right click on the component project folder and select new Component. Give it a name (in our case tFileInputCSVFilter) and a long name (File Input CSV Filter). Click "Next"
1. Select the required Jet files. In our case we need begin, main and end files. Select Next
1. You can now customize your component XML but we will leave that for later (editing the XML directly is a more poweful approach). Click on Finish. Below the Project Component folder our newly created component shows up
1. Right click on the component and select "Push Components to Palette". Click "OK"
1. Switch the perspective to "Data Integration" (Use "Window|Perspective" if there is no icon for this perspective on the right corner)
1. Create a new job and just name it "csv". We will use this job to test our CSVFilter
1. Search in the Palette (Window|Show View|Palette) for the component. You can now drag and drop it in the job canvas
1. Nothing fancy so far. You can run your job but nothing really will happen. You just got a lot of scaffold files to work with. It is time to get dirty ...</li>

Customizing the scaffold files
==============================
1. Go to the "Component Designer" perspective
1. Go to the component designer perspective. Drag a custom png file (32x32 pixels) with the proper name and drop it in the component, say yes to replace the existing one
1. Edit the XML following these directions. Note that code completion has apparently a problem when editing XML ( http://www.talendforge.org/forum/viewtopic.php?pid=97143 ). 
 * HEADER[@AUTHOR, @RELEASE_DATE, @VERSION]
 * HEADER[@STARTABLE]: For Startable we use true since our component will need no input from any other component
 * FAMILY: to define a path for your component (File/Input in our case). Talend will list your component below the directory structure you define here
 * CONNECTORS: to reflect how your component will work. In our case we use FLOW, and MAX_INPUT="1" (default generated values) but we also use SUBJOB_OK, SUBJOB_ERROR, COMPONENT_OK, COMPONENT_ERROR and RUN_IF connectors</li>
 * PARAMETERS: to define your custom settings. In our case we need INPUT_FILE, DELIMITER, LOOKUP_COLUMN, LOOKUP_VALUE all type TEXT. Note from the resulting project how we add the DEFAULT node for each of them
Side Note: Using the official eclipse distribution (not the Talend eclipse distribution) you could generate the component XML out of Component.xsd (the file is located inside the Talend distribution folder so just use find command) from "Navigator" view using "Generate|XML". This could be useful in case you are generating quick prototypes out of the component perspective (namely scaffolds from command line ;-)
1. Add labels in *_message.properties file for parameters (For example INPUT_FILE.NAME=INPUT_FILE)
1. Right click the component and select "Push Components to Palette". Click "OK". Note that if you are developing a component which has already been installed conflicts will be difficult to debug. Just remove the one you have been using from the file system to push the one you are developing from the custom components directory.
1. Nothing fancy yet, just your new icon showing up but we are ready to write Java code (or close to it)

Writing code
============
It is now time to change code in the begin, end and main templates to follow your logic needs. In our case we are just filtering the content of the CSV file as explained. For that we will use a library called "Java CSV" which is included in Talend distribution because in fact that is the library all components that process CSV use.

1. Go to the "Component Designer" perspective
1. Modify the xml file changing the empty CODEGENERATION node with:
<pre>
  &lt;CODEGENERATION&gt;
		  &lt;IMPORTS&gt;
				&lt;IMPORT NAME=&quot;JavaCSV&quot; MODULE=&quot;javacsv.jar&quot;
					REQUIRED=&quot;true&quot; /&gt;
			&lt;/IMPORTS&gt;
	&lt;/CODEGENERATION&gt;
</pre>
Note that I use as name "JavaCSV". In the existing components for Talend you can see they would name this import as "CsvReader" which IMO is miss leadng, the name here is just an identifier for the whole jar and not really related to importing a particular class name.
1. Open the begin, main and end templates. begin and end are executed once while main is there for looping. If you are familiar with JSP you are probably starting to get scared (if you are old enough you would remember JSP model 1 websites full of scriptlets. Yes I know you are already saying this must not be good but hey cheer up Talend Open Source is a great idea and after all you have all these Java Developers in your crew that could develop ETL components for you!)
1. See below the code for the begin template. The cid variable allows components to be reused in the same job. If you do not use the cid variable there will be naming clashing or unexpected behavior so be sure you understand what it does. The code that goes like scriptlets (inside the <%...%> access internal to talend artifacts like for example parameters). The code outside the scriptlets is necessary so the three template files can share variables. Talend takes the three template files and fills a section for that component in a final class per job. That means all variable as job scoped as you can certainly see when you switch from "Designer" to "Code" in the "Data Integration" perspective. Variables from the scriptlet portion can be used in the template section as you would do in JSP. All variables in the scriptlet portion must use the cid to avoid clashing. Scriptlets get messy in JSP and it is not any different in javalet (The talend template for begin, main, end sections). You can see below how we open a brace for the while loop. We close that brace in the end template and all operations to be performed inside the loop (you guessed it) are in the main template. Again note these are templates (like JSP files are) so the custom code is inserted like if it were HTML in a JSP file. As with JSP if you need to use inside your template code that is defined in the scriptlet section (the one between the percentage signs) you need to use the percentage signs as well.
<pre>
	<%@ jet 
		imports="
			org.talend.core.model.process.INode 
			org.talend.core.model.process.ElementParameterParser 
			org.talend.core.model.metadata.IMetadataTable 
			org.talend.core.model.metadata.IMetadataColumn 
			org.talend.core.model.process.IConnection
			org.talend.core.model.process.IConnectionCategory
			org.talend.designer.codegen.config.CodeGeneratorArgument
			org.talend.core.model.metadata.types.JavaTypesManager
			org.talend.core.model.metadata.types.JavaType
			java.util.List 
	    	java.util.Map		
		" 
	%>
	<% 
	    CodeGeneratorArgument codeGenArgument = (CodeGeneratorArgument) argument;
	    INode node = (INode)codeGenArgument.getArgument();
	    String cid = node.getUniqueName();	
	    String delimiter = ElementParameterParser.getValue(node, "__DELIMITER__");
		String inputFile = ElementParameterParser.getValue(node, "__INPUT_FILE__");
		String lookupValue = ElementParameterParser.getValue(node, "__LOOKUP_VALUE__");
		String lookupColumn = ElementParameterParser.getValue(node, "__LOOKUP_COLUMN__");
	%>
	String lookupValue<%=cid %> = <%=lookupValue %>;
	String lookupColumn<%=cid %> = <%=lookupColumn %>;
	com.csvreader.CsvReader csvReader<%=cid %> = new com.csvreader.CsvReader(<%=inputFile %>);
	csvReader<%=cid %>.setDelimiter(<%=delimiter %>.charAt(0));
	csvReader<%=cid %>.readHeaders();
	while (csvReader<%=cid %>.readRecord()) { 
</pre>
1. See below the main template. Note the imports again as well as the definition of the now well known cid variable. See how we use variables defined before in the template section. The code is trivial but certainly the use of cid variable all over the places gets annoying. Well, learn to live with it is all I can say at the moment.
<pre>
	<%@ jet 
		imports="
			org.talend.core.model.process.INode 
			org.talend.core.model.process.ElementParameterParser 
			org.talend.core.model.metadata.IMetadataTable 
			org.talend.core.model.metadata.IMetadataColumn 
			org.talend.core.model.process.IConnection
			org.talend.core.model.process.IConnectionCategory
			org.talend.designer.codegen.config.CodeGeneratorArgument
			org.talend.core.model.metadata.types.JavaTypesManager
			org.talend.core.model.metadata.types.JavaType
			java.util.List 
	    	java.util.Map		
		" 
	%>
	<% 
	    CodeGeneratorArgument codeGenArgument = (CodeGeneratorArgument) argument;
	    INode node = (INode)codeGenArgument.getArgument();
	    String cid = node.getUniqueName();	
	%>   
String value<%=cid %> = csvReader<%=cid %>.get(lookupColumn<%=cid %>);
if(value<%=cid %>.equals(lookupValue<%=cid %>)) {
  System.out.println(csvReader<%=cid %>.getRawRecord());
}
</pre>
1. Finally here is our end template which just closes the loop and the resource (file). Note again the imports and the code to define the now infamous cid variable:
<pre>
	<%@ jet 
		imports="
			org.talend.core.model.process.INode 
			org.talend.core.model.process.ElementParameterParser 
			org.talend.core.model.metadata.IMetadataTable 
			org.talend.core.model.metadata.IMetadataColumn 
			org.talend.core.model.process.IConnection
			org.talend.core.model.process.IConnectionCategory
			org.talend.designer.codegen.config.CodeGeneratorArgument
			org.talend.core.model.metadata.types.JavaTypesManager
			org.talend.core.model.metadata.types.JavaType
			java.util.List 
	    	java.util.Map		
		" 
	%>
	<% 
	    CodeGeneratorArgument codeGenArgument = (CodeGeneratorArgument) argument;
	    INode node = (INode)codeGenArgument.getArgument();
	    String cid = node.getUniqueName();	
	%>
	}
	csvReader<%=cid %>.close();
</pre>
1. Now re-push the component as explained before or simply regenerate the code using CTRL+Shift+F3 (In a MAC add Fn to the soup). If you get errors about not finding classes for example in our case CsvReader you might have to manually open the javacsv.jar file as a module. This is done from the modules perspective clicking on the jar symbol

Running the sample job
======================
1. Create a /tmp/sample.csv with the below content (which is similar to the one we used when using tJavaFlex in my original blog):
	person, city
	Paul, Miami
	John, Boston
	Mathew, San Francisco
	Craig, Miami
1. Switch to "Data Integration" perspective. If you haven't done it yet drop the component in your "csv" job canvas and press run. By default it will not filter anything and the 4 records will show up
1. Now in the basic settings of the component use for column "city" and for value "Miami" and you should get just one record
1. Run the job and only two records will show up 

Debugging
=========
If you are old enough to know how difficult was to debug JSP in its beginning when no IDE could do much for you then you will realize you are close that situation once again. Here are some hints though:

1. Any potential problems will be reported in the Workspace log which you can see from the error log view ("Window|Shoe View|General Error Log"). Alternatively you can tail the log file directly from $TALEND_WORKSPACE/.metadata/.log/. Please note that the $TALEND_WORKSPACE depends on your settings. By default Talend tries to use a directory called workspace in the home installation directory but you might have changed that.
1. Debugging compilation problems is difficult. Many times the error simply just says there is an error, period. So be careful on validating the XML against the XSD. Be also careful about where to put your code, remember shared across templates code goes outside the scriptlet tags and variables must use the cid variable.
2. Switch from "Designer" to "Code" in the job which includes the component and inspect error messages inside (they are marked in red as usual in the Eclipse environment). 


FAQ
===
1. Why I used github
I could have used Talend Exchange ( http://www.talendforge.org/exchange/ ) to host the component however I feel github and other git free repos have a tremendous advantage: Just forking components it is guaranteed you only get more and more features and bug fixes available for releasing. I believe Talend should consider moving to github as well.

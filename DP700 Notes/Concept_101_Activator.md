---


---

<h2 id="what-is-activator">What is Activator?</h2>
<p>Activator is Microsoft Fabric’s event detection and rules engine within Real-Time Intelligence.</p>
<ol>
<li>To use Activator, first you connect to real-time data sources.</li>
<li>Then you create  <strong>rules</strong>  to check for specific conditions, and when those conditions are met, Activator executes  <strong>actions</strong>  like sending email alerts, posting Teams messages, starting Power Automate flows, or running Fabric notebooks.</li>
</ol>
<h3 id="real-world-scenarios">Real-world scenarios</h3>
<p>Activator is useful in scenarios where timely responses matter:</p>
<ul>
<li><strong>Manufacturing operations</strong>  can automatically alert maintenance teams when equipment temperatures exceed safe operating ranges</li>
<li><strong>Supply chain managers</strong>  can be notified when shipments deviate from planned routes or experience unexpected delays</li>
<li><strong>Retail managers</strong>  can trigger inventory reorders when stock levels fall below critical thresholds</li>
<li><strong>IT operations teams</strong>  can automatically restart services when performance metrics indicate system degradation</li>
<li><strong>Financial institutions</strong>  can flag unusual transaction patterns for immediate review</li>
<li><strong>Healthcare facilities</strong>  can alert staff when patient monitoring devices detect critical changes</li>
</ul>
<h2 id="configure-activator-for-your-data">Configure Activator for your data</h2>
<p>There are several ways to configure Activator for monitoring your data. One approach uses <strong>business objects</strong>.</p>
<h3 id="understand-business-objects">Understand business objects</h3>
<p>Business objects represent the entities you want to monitor. For a package delivery company, each package becomes an  <strong>object</strong>  in Activator. Each object has  <strong>properties</strong>  which are the specific data points you want to monitor.</p>
<p>Here’s how this organization works:</p>
<ul>
<li><strong>Objects</strong>  represent individual instances (Package001, Package002, Package003)</li>
<li><strong>Properties</strong>  represent the data attributes for each instance (Temperature, City, DeliveryState, HoursInTransit)</li>
<li><strong>Events</strong>  contain the actual data values that flow in from your data sources</li>
</ul>
<p>Your data sources send events containing information for multiple packages. Activator uses this incoming data to automatically update the property values for each package object.</p>
<h3 id="create-objects-from-eventstreams">Create objects from eventstreams</h3>
<p>When you configure Activator as a destination for an eventstream, you can then create business objects in Activator. During this setup, you tell Activator which data fields represent objects and which fields are properties to evaluate.</p>
<p>Here’s how you create objects from an eventstream:</p>
<ol>
<li><strong>Configure Activator as a destination</strong>  - In an eventstream</li>
<li><strong>Open the Activator</strong>  - Then select the option to create a new object</li>
<li><strong>Choose your unique identifier</strong>  - Select the field that uniquely identifies the object (like PackageId or DeviceID)</li>
<li><strong>Select properties</strong>  - Choose which data fields you want to monitor as properties</li>
</ol>
<p>For example, if your eventstream contains package delivery data with fields like  <code>PackageId</code>,  <code>Temperature</code>,  <code>City</code>,  <code>DeliveryState</code>, and  <code>HoursInTransit</code>, you might:</p>
<ul>
<li>Use  <code>PackageId</code>  as your unique identifier to create separate package objects</li>
<li>Select  <code>Temperature</code>  and  <code>HoursInTransit</code>  as properties you want to monitor</li>
<li>Let Activator automatically create objects for each unique package as data flows in</li>
</ul>
<p><img src="https://learn.microsoft.com/en-in/training/wwl/use-fabric-activator/media/create-object.png" alt="Activator interface for creating a new object"></p>
<p><em>Once configured, your eventstream continuously feeds data to Activator. New events update existing object properties, new unique identifiers automatically create new objects, and property values always reflect the most recent data from your stream. This configuration means your rules can evaluate conditions as soon as new data arrives.</em></p>
<h3 id="alternative-alerting-approaches">Alternative alerting approaches</h3>
<p>The Activator engine also supports creating alerts. You can create dashboard alerts directly from Real-Time Dashboard visualizations, set up system event alerts to monitor Fabric workspace activities and OneLake file operations, or create query alerts from KQL Queryset results and visualizations. These approaches use the same Activator engine but interpret data differently than the business objects model.</p>
<h2 id="create-rules-in-activator">Create rules in Activator</h2>
<p>Rules define the conditions you want to detect on your objects and the actions to take when those conditions are met.</p>
<p>Imagine you’re managing a package delivery company with temperature-sensitive medicines. Each package has sensors sending temperature and location data to your Eventstream. You need to catch temperature problems before they damage valuable cargo.</p>
<p>Let’s walk through creating a rule for this scenario. When you create a rule in the Activator interface, a  <em>Definition</em>  pane opens with several sections to configure. We’ll focus on the first three sections in the  <em>Definition</em>  pane now - Monitor, Condition, and Property filter - and cover Actions in the next unit.</p>
<p><img src="https://learn.microsoft.com/en-in/training/wwl/use-fabric-activator/media/activator-definition-pane.png" alt="Definition pane with Monitor, Condition, and Property filter sections for configuring a temperature rule."></p>
<h3 id="choose-what-to-monitor">Choose what to monitor</h3>
<p>The  <strong>Monitor</strong>  section is where you configure what Activator watches. This section directs Activator to monitor specific data properties.</p>
<p>First, you select an  <strong>Attribute</strong>  - the specific property from your event data. For our package delivery scenario, you’d choose Temperature from your package sensor data.</p>
<p>Raw temperature readings can be noisy. A package might show brief spikes when moved or dips when passing through different environments. That’s where  <strong>Summarization</strong>  becomes crucial for seeing the bigger picture:</p>
<ul>
<li><strong>Average</strong>  - Smooth out the noise by averaging readings over time</li>
<li><strong>Minimum/Maximum</strong>  - Catch the extreme values that matter most</li>
<li><strong>Count</strong>  - Track how many readings you’re getting (useful for detecting sensor failures)</li>
<li><strong>Total</strong>  - Add up values when you’re counting events rather than measuring levels</li>
</ul>
<p>When you add summarization, you control two timing settings:</p>
<ul>
<li><strong>Window size</strong>: How much historical data to include (maybe 10 minutes of temperature readings)</li>
<li><strong>Step size</strong>: How often to recalculate (perhaps every 5 minutes for near real-time monitoring)</li>
</ul>
<p>For example, instead of reacting to every single temperature reading, you might monitor the average temperature over 10-minute windows, updating every 5 minutes. This approach catches sustained problems while ignoring brief fluctuations.</p>
<h3 id="define-when-to-act">Define when to act</h3>
<p>The  <strong>Condition</strong>  section sets your execution point - when Activator should spring into action.</p>
<p>You choose from different detection approaches:</p>
<ul>
<li><strong>Threshold monitoring</strong>: Alert when values cross your safety limits (Temperature  <strong>is greater than</strong>  68°F)</li>
<li><strong>Change detection</strong>: Monitor trends (Temperature  <strong>increases above</strong>  baseline)</li>
<li><strong>Range monitoring</strong>: Track entry/exit from safe zones</li>
<li><strong>Missing data</strong>: Catch sensor failures (<strong>No new events for more than</strong>  30 minutes)</li>
</ul>
<p>You also set the  <strong>Value</strong>  (your threshold like 68°F) and  <strong>Occurrence</strong>  behavior:</p>
<ul>
<li><strong>Every time</strong>  for immediate alerts</li>
<li><strong>When it has been true for</strong>  to identify persistent problems (stays high for 15 minutes)</li>
</ul>
<h3 id="focus-your-scope">Focus your scope</h3>
<p>You might want to apply your rule only to specific events, not every event in your stream. The  <strong>Property filter</strong>  section lets you narrow down which events get evaluated.</p>
<p>Maybe you only want to monitor:</p>
<ul>
<li>Cold chain packages:  <strong>ColdChainType equals</strong>  “medicine”</li>
<li>High-priority routes:  <strong>City equals</strong>  “Seattle”</li>
<li>Warm shipments:  <strong>Temperature is greater than</strong>  68°F</li>
</ul>
<p>You can combine up to three filters to create precise targeting.</p>
<p>With your rule defined, the next step is configuring what actions Activator should take when the rule conditions are met.</p>
<h2 id="configure-actions-in-activator">Configure actions in Activator</h2>
<p>Actions are what happens when your rule conditions are met. They complete your Activator setup, turning condition evaluation into notifications and automated workflows. Activator offers four types of actions.</p>
<h3 id="email-actions">Email actions</h3>
<p>Email actions send detailed information that people can review and respond to within hours or days. Use email when the recipient needs comprehensive context or when immediate response isn’t critical.</p>
<h3 id="teams-actions">Teams actions</h3>
<p>Teams actions send immediate messages to channels or individuals. Use Teams when you need quick responses and team coordination. Teams notifications appear immediately and let teams discuss and coordinate their response.</p>
<h3 id="power-automate-actions">Power Automate actions</h3>
<p>Power Automate is Microsoft’s workflow automation service that connects different apps and services together. Power Automate actions execute multi-step business processes across different systems - automatically performing a series of tasks that would normally require manual steps across multiple applications.</p>
<h3 id="fabric-item-actions">Fabric item actions</h3>
<p>Fabric item actions execute data pipelines or notebooks for other processing and analysis. Use Fabric items when you need to process more data or perform advanced analysis in response to evaluated conditions.</p>
<p>With data sources connected, rules defined, and actions configured, you have everything needed for Activator to automatically take action when conditions occur in your streaming data.</p>


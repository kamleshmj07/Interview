---


---

<h1 id="microsoft-fabric-licenses">Microsoft Fabric licenses</h1>
<p>Microsoft Fabric licenses and capacities determine how users create, share, and view items across your organization.</p>
<p>The following diagram illustrates two common ways organizations can structure their Microsoft Fabric deployment using tenants and capacities.</p>
<p><img src="https://learn.microsoft.com/en-us/fabric/enterprise/media/licenses/tenants-capacities.png#lightbox" alt="Screenshot of two organizational deployment examples: Org A with one tenant and three capacities; Org B with two tenants each containing several capacities and workspaces."></p>
<p>For example, a company can use a single Microsoft Entra tenant for all business units, or separate tenants for different divisions or compliance needs. Each tenant can have one or more Fabric capacities, often aligned to geographic or business requirements.</p>
<h2 id="core-building-blocks">Core building blocks</h2>
<p>This section describes tenants, capacities, and workspaces, which are helpful in understanding a Fabric deployment.</p>
<h3 id="tenant">Tenant</h3>
<p>Microsoft Fabric runs in a Microsoft Entra tenant. A tenant is associated with one primary DNS domain, and you can add additional custom domains to it.</p>
<p>If you don’t already have a tenant, one is created automatically when you acquire a free, trial, or paid Microsoft online service license, or you can add your domain to an existing tenant.</p>
<p>After the tenant exists, add one or more Fabric capacities to support workloads.</p>
<p>To create a tenant manually, see  <a href="https://learn.microsoft.com/en-us/entra/fundamentals/create-new-tenant">Quickstart: Create a new tenant in Microsoft Entra ID</a>.</p>
<h3 id="capacity">Capacity</h3>
<p>A Microsoft Fabric capacity resides in a tenant. Each capacity in a tenant is a distinct resource pool for Microsoft Fabric. The size of the capacity determines the amount of computation power available.</p>
<p>Your capacity lets you:</p>
<ul>
<li>
<p>Use all Microsoft Fabric features licensed by capacity</p>
</li>
<li>
<p>Create Microsoft Fabric items and connect to other Microsoft Fabric items</p>
</li>
</ul>
<blockquote>
<p>Note:     To create Power BI items in workspaces that aren’t  <em>My<br>
workspace</em>, you need a  <em>Pro</em>  license.</p>
</blockquote>
<ul>
<li>Save your items to a workspace and share them with a user that has an appropriate license.</li>
</ul>
<p>Capacities use stock-keeping units (SKUs).</p>
<p>Each SKU provides Fabric resources for your organization. Your organization can have as many capacities as needed.</p>
<p>The table lists the Microsoft Fabric SKUs. Capacity units (CUs) measure the compute power for each SKU.</p>
<p>For customers familiar with Power BI, the table also lists Power BI Premium per capacity <em>P</em> SKUs and virtual cores (v-cores).</p>
<p>Power BI Premium <em>P</em> SKUs support Microsoft Fabric. <em>A</em> and <em>EM</em> SKUs only support Power BI items.</p>
<p>This table is provided as a reference for comparing compute capacity and should not be interpreted as functional or licensing equivalence.</p>
<p><img src="https://www.tigeranalytics.com/wp-content/uploads/2024/07/MSFabric_Infographics_4.jpg" alt="Microsoft Fabric"></p>
<p><img src="https://www.tigeranalytics.com/wp-content/uploads/2024/07/MSFabric_Infographics_6.jpg" alt="Microsoft Fabric"></p>
<p>Learn More:<br>
<a href="https://www.tigeranalytics.com/perspectives/blog/a-comprehensive-guide-to-pricing-and-licensing-on-microsoft-fabric/">Tiger Analytics - Fabric Licensing</a></p>
<h3 id="workspace">Workspace</h3>
<p>Workspaces reside within capacities and are used as containers for Microsoft Fabric items.</p>
<p>Each Microsoft Entra tenant with Fabric has a shared capacity that hosts all <em>My Workspaces</em> and workspaces using the Power BI Pro or Power BI Premium Per-User(PPU) workspace types.<br>
By default, workspaces are created in your tenant’s shared capacity. When your tenant has other capacities, assign any workspace—including <em>My Workspaces</em>—to any capacity in the tenant.</p>
<p>A Power BI Premium Per User (PPU) workspace type isn’t a Fabric capacity; it uses a shared Premium feature pool and doesn’t itself enable Fabric (non–Power BI) item creation unless an F capacity also exists.</p>
<p><strong>Capacity-Based Licensing</strong>: This licensing model is required for operating Fabric’s services, where Capacity Units (CUs) define the extent of compute resources available to your organization. Different Stock Keeping Units (SKUs) are designed to accommodate varying workload demands, ranging from F2 to F2048. This flexibility allows businesses to scale their operations up or down based on their specific needs.</p>
<p><strong>Per-User Licensing</strong>: User-based licensing was used in Power BI, and this has not changed in Fabric (for compatibility). The User accounts include:</p>
<ul>
<li>Free</li>
<li>Pro</li>
<li>Premium Per User (PPU)</li>
</ul>
<p><img src="https://www.tigeranalytics.com/wp-content/uploads/2024/07/MSFabric_Infographics_09.png" alt="Microsoft Fabric"></p>
<p>Learn More:<br>
<a href="https://learn.microsoft.com/en-us/fabric/enterprise/licenses">MS Fabric Licensing</a></p>


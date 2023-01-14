```

namespace Microsoft.AzSK.ATS.ADOScanner.Repositories
{
    using System;
    using System.Collections.Concurrent;
    using System.Collections.Generic;
    using System.Globalization;
    using System.Linq;
    using System.Threading.Tasks;
    using Microsoft.AzSK.ATS.ADOScanner.Models;
    using Microsoft.AzSK.ATS.Core;
    using Microsoft.AzSK.ATS.Extensions.Models;
    using Microsoft.Extensions.Logging;
    using Microsoft.TeamFoundation.Build.WebApi;
    using Newtonsoft.Json.Linq;

    /// <summary>
    /// Repository for build resources.
    /// </summary>
    public class BuildAPIRepository : ADORepositoryBase
    {
        private readonly string _buildDefinitions = "/{0}/{1}/_apis/build/definitions?api-version=6.0&queryOrder=DefinitionNameAscending";
        private readonly string _buildDefinitionsWithIds = "/{0}/{1}/_apis/build/definitions?api-version=6.0&queryOrder=DefinitionNameAscending&definitionIds={2}";
        private readonly string _buildDefinition = "/{0}/{1}/_apis/build/definitions/{2}?api-version=6.0";
        private int maxDegreeOfParallelismForBuildDefinition;

        /// <summary>
        /// Get a list of build resources using REST API.
        /// </summary>
        /// <returns>List of BuildResource object.</returns>
        /// <typeparam name="T">Placeholder for generic type.</typeparam>
        public override List<T> FetchResources<T>()
        {
            Log.LogTrace(EventIds.RequestStarting, "Getting build details for {Project}", WorkItemInstance.ProjectIdentifierUrl);

            // Step1: Get list of builds
            var buildListUrl = GetBuildDefinitionUrl();
            List<BuildResource> buildResources = new List<BuildResource>();

            buildResources = ADOHttpClientHelper.GetWithNextLink(buildListUrl, WorkItemInstance.WorkItem.Id )
                        .Select(a => new BuildResource(WorkItemInstance, a.ToObject<Models.BuildDefinitionReference>())).ToList();

            ConcurrentBag<BuildResource> buildConcurrentList = new ConcurrentBag<BuildResource>();

            maxDegreeOfParallelismForBuildDefinition = 1;
            if (RepositorySettings.Build.TryGetValue("MaxDegreeOfParallelismForBuildDefinition", out string maxParallelLimit))
            {
                maxDegreeOfParallelismForBuildDefinition = int.Parse(maxParallelLimit);
            }

            // Step2: Get individual build definition details. Adjust degreee of parallelism based on build API throttling limits observations.
            Parallel.ForEach(buildResources, new ParallelOptions() { MaxDegreeOfParallelism = maxDegreeOfParallelismForBuildDefinition }, buildResource =>
            {
                try
                {
                    var buildDefinitionUrl = string.Format(_buildDefinition, WorkItemInstance.OrganizationName, WorkItemInstance.ProjectName, buildResource.Instance.Id.ToString());
                    buildResource.BuildDefinition = new Models.BuildDefinition();
                    JObject builddefn = JObject.Parse(ADOHttpClientHelper.Get(buildDefinitionUrl, WorkItemInstance.WorkItem.Id));
                    buildResource.BuildDefinition = builddefn?.ToObject<Models.BuildDefinition>();
                    FetchTaskGroups(buildResource, builddefn);
                    buildConcurrentList.Add(buildResource as BuildResource);
                }
                catch (Exception ex)
                {
                    Log.LogError(ex, "Error in build definition fetch: {identifier}", buildResource.Instance.Url);
                }
            });

            Log.LogTrace(EventIds.RequestStarting, "Completed getting build details for {Project}", WorkItemInstance.ProjectIdentifierUrl);
            return buildConcurrentList.ToList().Cast<T>().ToList();
        }

        /// <summary>
        /// Fetch all task groups used in build definition.
        /// </summary>
        /// <param name="resource">Build resource inventory object.</param>
        /// <param name="buildDefnObj">Build definition API response.</param>
        public void FetchTaskGroups(BuildResource resource, JObject buildDefnObj)
        {
            ConcurrentBag<TaskGroup> taskGroups = new ConcurrentBag<TaskGroup>();

            // Getting task groups used in build only if the pipeline is classic and is not empty
            // YAML pipeline does not have phases attribute
            if (buildDefnObj["process"]["phases"] != null && buildDefnObj["process"]["phases"][0]["steps"] != null)
            {
                Parallel.ForEach(buildDefnObj["process"]["phases"][0]["steps"], step =>
                {
                    if (step["task"]["definitionType"].Value<string>() != null && step["task"]["definitionType"].Value<string>().Equals("metaTask"))
                    {
                        var taskGrpId = step["task"]["id"].Value<string>();
                        Guid taskGrpGuid = new Guid(taskGrpId);
                        TaskGroup taskGroup = new TaskGroup();
                        taskGroup.Id = taskGrpGuid;
                        taskGroup.TaskName = step["displayName"].Value<string>();
                        taskGroups.Add(taskGroup);
                    }
                });
            }

            resource.BuildDefinition.TaskGroupList = taskGroups.ToList();
        }

        private string GetBuildDefinitionUrl()
        {
            // Check if this is a selective scan. If so then add resource id parameters to the URL.
            var buildIdList = WorkItemInstance.ResourceFilter?
                                        .Where(resource => resource.Type == nameof(ScannedResourceLabels.BuildDefinition))
                                        .FirstOrDefault()?
                                        .Ids;

            var buildListURL = string.Empty;

            if (buildIdList == null || buildIdList.Count == 0)
            {
                buildListURL = string.Format(
                            CultureInfo.InvariantCulture,
                            _buildDefinitions,
                            WorkItemInstance.OrganizationName,
                            WorkItemInstance.ProjectName);
            }
            else
            {
                var buildIds = string.Join(",", buildIdList);
                buildListURL = string.Format(
                            CultureInfo.InvariantCulture,
                            _buildDefinitionsWithIds,
                            WorkItemInstance.OrganizationName,
                            WorkItemInstance.ProjectName,
                            buildIds);
            }

            return buildListURL;
        }
    }
}

```

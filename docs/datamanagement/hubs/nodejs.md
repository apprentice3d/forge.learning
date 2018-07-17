# List hubs & projects

## datamanagement.js

Create a `/server/datamanagement.js` file with the following content:

```javascript
'use strict';

// web framework
var express = require('express');
var router = express.Router();

// Forge NPM
var forgeSDK = require('forge-apis');

// actually perform the token operation
var oauth = require('./oauth');

// Create a new bucket 
router.get('/api/forge/datamanagement', function (req, res) {
    // the id querystring parameter contains what was selected
    // on the UI tree, make sure it's valid
    var href = decodeURIComponent(req.query.id);
    if (href === '') {
        res.status(500).end();
        return;
    }

    // get the access token
    var credentials = new oauth(req.session);
    credentials.getTokenInternal().then(function (tokenInternal) {
        // if href == hash, then it's the root tree node
        if (href === '#') {
            getHubs(credentials.OAuthClient(), tokenInternal, res);
        }
        else {
            // otherwise let's break it by / 
            var params = href.split('/');
            var resourceName = params[params.length - 2];
            var resourceId = params[params.length - 1];

            // what was selected?
            switch (resourceName) {
                case 'hubs':
                    getProjects(resourceId, credentials.OAuthClient(), tokenInternal, res);
                    break;
                case 'projects':
                    // for a project, first we need the top/root folder
                    var hubId = params[params.length - 3];
                    getFolders(hubId, resourceId/*project_id*/, credentials.OAuthClient(), tokenInternal, res)
                    break;
                case 'folders':
                    var projectId = params[params.length - 3];
                    getFolderContents(projectId, resourceId/*folder_id*/, credentials.OAuthClient(), tokenInternal, res);
                    break;
                case 'items':
                    var projectId = params[params.length - 3];
                    getVersions(projectId, resourceId/*item_id*/, credentials.OAuthClient(), tokenInternal, res);
                    break;
            }
        }
    })
    .catch(function(){
        res.status(401).end();
    })
});
```

The above receives the request from the UI tree. The `id` parameter indicates the node that is being expanded: `#` means root node, so list hubs. After that it contains the `href` of the resource, so when expanding one `hub` the endpoint should return the projects for the hub. The above code calls different `get` functions. To complete it, also copy the following content to the file:

```javascript
function getHubs(oauthClient, credentials, res) {
    var hubs = new forgeSDK.HubsApi();
    hubs.getHubs({}, oauthClient, credentials)
        .then(function (data) {
            var hubsForTree = [];
            data.body.data.forEach(function (hub) {
                var hubType;

                switch (hub.attributes.extension.type) {
                    case "hubs:autodesk.core:Hub":
                        hubType = "hubs";
                        break;
                    case "hubs:autodesk.a360:PersonalHub":
                        hubType = "personalHub";
                        break;
                    case "hubs:autodesk.bim360:Account":
                        hubType = "bim360Hubs";
                        break;
                }

                hubsForTree.push(prepareItemForTree(
                    hub.links.self.href,
                    hub.attributes.name,
                    hubType,
                    true
                ));
            });
            res.json(hubsForTree);
        })
        .catch(function (error) {
            console.log(error);
            res.status(500).end();
        });
}

function getProjects(hubId, oauthClient, credentials, res) {
    var projects = new forgeSDK.ProjectsApi();

    projects.getHubProjects(hubId, {}, oauthClient, credentials)
        .then(function (projects) {
            var projectsForTree = [];
            projects.body.data.forEach(function (project) {
                var projectType = 'projects';
                switch (project.attributes.extension.type) {
                    case 'projects:autodesk.core:Project':
                        projectType = 'a360projects';
                        break;
                    case 'projects:autodesk.bim360:Project':
                        projectType = 'bim360projects';
                        break;
                }

                projectsForTree.push(prepareItemForTree(
                    project.links.self.href,
                    project.attributes.name,
                    projectType,
                    true
                ));
            });
            res.json(projectsForTree);
        })
        .catch(function (error) {
            console.log(error);
            res.status(500).end();
        });
}

function getFolders(hubId, projectId, oauthClient, credentials, res) {
    var projects = new forgeSDK.ProjectsApi();
    projects.getProjectTopFolders(hubId, projectId, oauthClient, credentials)
        .then(function (topFolders) {
            var folderItemsForTree = [];
            topFolders.body.data.forEach(function (item) {
                folderItemsForTree.push(prepareItemForTree(
                    item.links.self.href,
                    item.attributes.displayName == null ? item.attributes.name : item.attributes.displayName,
                    item.type,
                    true
                ))
            });
            res.json(folderItemsForTree);
        })
        .catch(function (error) {
            console.log(error);
            res.status(500).end();
        });
}

function getFolderContents(projectId, folderId, oauthClient, credentials, res) {
    var folders = new forgeSDK.FoldersApi();
    folders.getFolderContents(projectId, folderId, {}, oauthClient, credentials)
        .then(function (folderContents) {
            var folderItemsForTree = [];
            folderContents.body.data.forEach(function (item) {
                var name = (item.attributes.name == null ? item.attributes.displayName : item.attributes.name);
                if (name !== '') { // BIM 360 Items with no displayName also don't have storage, so not file to transfer
                    folderItemsForTree.push(prepareItemForTree(
                        item.links.self.href,
                        name,
                        'items',
                        true
                    ));
                }
            });
            res.json(folderItemsForTree);
        })
        .catch(function (error) {
            console.log(error);
            res.status(500).end();
        });
}

function getVersions(projectId, itemId, oauthClient, credentials, res) {
    var items = new forgeSDK.ItemsApi();
    items.getItemVersions(projectId, itemId, {}, oauthClient, credentials)
        .then(function (versions) {
            var versionsForTree = [];
            versions.body.data.forEach(function (version) {
                var dateFormated = new Date(version.attributes.lastModifiedTime).toLocaleString();
                var versionst = version.id.match(/^(.*)\?version=(\d+)$/)[2];
                var viewerUrn = (version.relationships != null && version.relationships.derivatives != null ? version.relationships.derivatives.data.id : null);
                versionsForTree.push(prepareItemForTree(
                    viewerUrn,
                    decodeURI('v' + versionst + ': ' + dateFormated + ' by ' + version.attributes.lastModifiedUserName),
                    (viewerUrn != null ? 'versions' : 'unsupported'),
                    false
                ));
            });
            res.json(versionsForTree);
        })
        .catch(function (error) {
            console.log(error);
            res.status(500).end();
        })
}

// format data for tree
function prepareItemForTree(_id, _text, _type, _children) {
    return { id: _id, text: _text, type: _type, children: _children };
}

module.exports = router;
```

The last `get` function returns the **Versions** for each item (file), where the `.relationships.derivatives.data.id` property contains the `URN` for the **Viewer**. It's important to test if this attribute is available as some items may not have viewables (e.g. a ZIP or DOCx file) or may not have being translated yet.

Note how we reuse the /server/oauth.js file to call .getTokenInternal() on all functions.

Next: [User information](oauth/user/readme)
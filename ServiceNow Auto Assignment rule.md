# ServiceNow Auto Assignemtn Business Rule


```javascsript
  (function executeRule(current, previous /*, gs, script_include*/ ) {

    // Define the name of the group for auto-assignment.
    var targetGroupName = 'SOC Incident Auto Assignment';

    var targetGroup = new GlideRecord('sys_user_group');
    if (!targetGroup.get('name', targetGroupName)) {
        gs.log('Auto-assign SOC Incidents: The target group "' + targetGroupName + '" was not found.', 'SOC Auto-Assignment');
        return;
    }
    var targetGroupID = targetGroup.sys_id;

    if (current.assignment_group == targetGroupID) {
        //return;
    }

    // 1. Get all active members of the target group (as sys_ids)
    var allActiveMembers = [];
    var grMember = new GlideRecord('sys_user_grmember');
    grMember.addQuery('group', targetGroupID);
    grMember.addQuery('user.active', true);
    grMember.query();
    while (grMember.next()) {
        allActiveMembers.push(grMember.user.toString());
    }

    if (allActiveMembers.length == 0) {
        gs.log('Auto-assign SOC Incidents: No active users were found in the "' + targetGroupName + '" group.', 'SOC Auto-Assignment');
        current.work_notes = 'Assignment to "' + targetGroupName + '" failed. The group has no active members.';
        return;
    }

    // 2. Convert the list of member sys_ids into a list of User ID strings
    var memberUserIDs = [];
    var userLookup = new GlideRecord('sys_user');
    userLookup.addQuery('sys_id', 'IN', allActiveMembers.join(','));
    userLookup.query();
    while (userLookup.next()) {
        memberUserIDs.push(userLookup.user_name.toString());
    }

    // 3. Find unique active sessions by checking the array before adding a user
    var loggedInUserIDs = [];
    var userSession = new GlideRecord('sys_user_session'); // Reverted to GlideRecord
    userSession.addEncodedQuery('invalidated=NULL');
    userSession.addQuery('name', 'IN', memberUserIDs.join(','));
    userSession.query();
    while (userSession.next()) {
        var userName = userSession.name.toString();
        // Only add the user to the list if they are not already in it
        if (!loggedInUserIDs.includes(userName)) {
            loggedInUserIDs.push(userName);
        }
    }

    // 4. Determine the final list of users (as sys_ids) to assign from
    var assignmentList;
    var workNoteMessage = '';

    if (loggedInUserIDs.length > 0) {
        // If logged-in users were found, convert their User IDs back to sys_ids for the assignment field
        var finalUserSysIds = [];
        var userReverseLookup = new GlideRecord('sys_user');
        userReverseLookup.addQuery('user_name', 'IN', loggedInUserIDs.join(','));
        userReverseLookup.query();
        while (userReverseLookup.next()) {
            finalUserSysIds.push(userReverseLookup.sys_id.toString());
        }
        assignmentList = finalUserSysIds;
        workNoteMessage = 'Auto-Assignment Triggered: Security Incident assigned to the least loaded logged-in user in the "' + targetGroupName + '" group: ';
    } else {
        // Fallback: If no logged-in users were found, use the original list of all active members
        assignmentList = allActiveMembers;
        workNoteMessage = 'Auto-Assignment Triggered: No users were logged-in within the group "' + targetGroupName + '" Security Incident assigned to the least loaded user ';
    }

    // // 5a. Perform Random assignment
    // var randomIndex = Math.floor(Math.random() * assignmentList.length);
    // var randomUserSysID = assignmentList[randomIndex];

    // // 5b. Perform "Least-Loaded" assignment
    var randomUserSysID = '';

    // If there's only one potential assignee, assign it directly.
    if (assignmentList.length == 1) {
        randomUserSysID = assignmentList[0];
    } else {
        // A. Create a map to store incident counts for each user, defaulting to 0.
        var userIncidentCounts = {};
        for (var i = 0; i < assignmentList.length; i++) {
            var userId = assignmentList[i];
            userIncidentCounts[userId] = 0;
        }

        // B. Use GlideAggregate to get the real counts of active incidents for our users.
        var incidentAgg = new GlideAggregate('sn_si_incident');
        incidentAgg.addQuery('active', true);
        incidentAgg.addQuery('assignment_group', targetGroupID);
        incidentAgg.addQuery('assigned_to', 'IN', assignmentList.join(','));
        incidentAgg.addAggregate('COUNT', 'assigned_to');
        incidentAgg.query();

        while (incidentAgg.next()) {
            var userSysId = incidentAgg.assigned_to.toString();
            var incidentCount = parseInt(incidentAgg.getAggregate('COUNT', 'assigned_to'), 10); // Get the user's incident count from the aggregate result and convert it from a string to an integer.
            userIncidentCounts[userSysId] = incidentCount;
        }

        // C. Find the minimum count and build a list of all users who are tied.
        var minIncidents = Infinity;
        var tiedUsers = [];

        for (var sysId in userIncidentCounts) {
            var currentCount = userIncidentCounts[sysId];
            if (currentCount < minIncidents) {
                // If we found a new minimum, start a new list of tied users
                minIncidents = currentCount;
                tiedUsers = [sysId];
            } else if (currentCount === minIncidents) {
                // If we found a user with the same minimum count, add them to the list
                tiedUsers.push(sysId);
            }
        }

        // D. Randomly select one user from the list of tied users.
        var randomIndex = Math.floor(Math.random() * tiedUsers.length);
        randomUserSysID = tiedUsers[randomIndex];
    }

    var assignedUserName = '';
    var userRecord = new GlideRecord('sys_user');
    if (userRecord.get(randomUserSysID)) {
        assignedUserName = userRecord.getDisplayValue('name');
    }

    // 6. Update the incident record
    current.setValue('assignment_group', targetGroupID);
    current.setValue('assigned_to', randomUserSysID);
    current.work_notes = workNoteMessage + assignedUserName + '.';
	current.update();

})(current, previous);
```

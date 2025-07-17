---
title: Obsidian 에서 Jira 를 Tasks 로 변환하기
created: 2025-07-16 00:00:00 +0900
layout: post
categories:
  - Obsidian
tags:
  - obsidian
  - jira
  - tasks
type: post
published: true
meta: {}
---
`Obsidian` 을 이용해서 일정을 관리하다보면 `Jira-issue Plugin`에 많은 불편함을 느낀다. 그 중 가장 불편한 것은 `Jira Issue`를 `Tasks` 처럼 관리하지 못한다는 것이다.
하지만 `Templater`의 `js`를 이용해 해당 불편함을 어느정도 해소할 수 있다.

```js
async function getJiraIssues(tp) {
	const jqlQuery = "status not in (''DONE', 'COMPLETE') AND assignee = currentUser() order by priority DESC";
	
	try {
		const searchResults = await $ji.base.getSearchResults(jqlQuery, {
		limit: 1000,
		fields: ['summary', 'priority', 'key', 'project', 'issuetype', 'reporter', 'assignee', 'status', 'labels', 'duedate']
	});
	
	const titleDate = tp.file.title.split(',')[0].trim();
	const noteDate = new Date(titleDate);
	if (isNaN(noteDate.getTime())) {
		throw new Error("Invalid date in the note title");
	}
	
	noteDate.setHours(0, 0, 0, 0);
	
	const endOfWeek = new Date(noteDate);
	endOfWeek.setDate(noteDate.getDate() + (6 - noteDate.getDay()));
	
	let beforeTasks = '';
	let todayTasks = '';
	let weekTasks = '';
	let noDueDateTasks = '';
	
	searchResults.issues.forEach(issue => {
		const priorityIcon = issue.fields.priority.iconUrl;
		let dueDate = issue.fields.duedate ? new Date(issue.fields.duedate) : null;
		let dueDateStr = 'No Due Date';
		
		const taskItem = `- [ ] <img src="${priorityIcon}" alt="우선순위 아이콘" style="vertical-align: middle; width: 15px; height: 15px;"> <span style="color: #000000; font-weight: 600;"><a href="https://your-domain.net/browse/${issue.key}">[${issue.key}]</a> ${issue.fields.summary}</span>`;
		
		if (!dueDate) {
			noDueDateTasks += `${taskItem} 📅 ${dueDateStr}\n`;
		} else {
		
		dueDate.setHours(0, 0, 0, 0);
		const year = dueDate.getFullYear();
		const month = String(dueDate.getMonth() + 1).padStart(2, '0');
		const day = String(dueDate.getDate()).padStart(2, '0');
		dueDateStr = `${year}-${month}-${day}`;
		
		const formattedTaskItem = `${taskItem} 📅 ${dueDateStr}\n`;
	
		if (dueDate < noteDate) {	
			beforeTasks += formattedTaskItem;
		} else if (dueDate.getTime() === noteDate.getTime()) {
			todayTasks += formattedTaskItem;
		} else if (dueDate <= endOfWeek) {
			weekTasks += formattedTaskItem;
		}
		}
	});

	let todoList = '';	
	todoList += '## Tasks' + '\n'
	
	if (beforeTasks) todoList += '### Before Tasks\n' + beforeTasks + '\n' + '---' + '\n';
	if (todayTasks) todoList += '### Today Tasks\n' + todayTasks + '\n' + '---' + '\n';
	if (weekTasks) todoList += '### Week Tasks\n' + weekTasks + '\n' + '---' + '\n';
	if (noDueDateTasks) todoList += '### Not DueDate Tasks\n' + noDueDateTasks;
		return todoList.trim();
	} catch (error) {
		return "오류가 발생했습니다: " + error.message;
	}
}

module.exports = getJiraIssues;
```
---
title: Automating GitLab Artifact Cleanup with Jenkins
slug: automating-gitlab-artifact-cleanup-with-jenkins
date_published: 2026-03-31T07:40:29.000Z
date_updated: 2026-03-31T07:40:29.000Z
excerpt: We automated GitLab artifact cleanup using a Python script and Jenkins, keeping recent jobs and deleting older artifacts to reclaim storage safely without affecting active pipelines.
---

In our organization, we use a self-hosted GitLab instead of the SaaS version. While this gives us full control, it also means that storage management becomes our responsibility. During my first month, I was tasked with reducing disk usage on GitLab, but with a constraint: I didn’t have access to the GitLab server directly. This meant no manual checks, only API access.

To identify which repositories were consuming the most storage, I wrote a Python script to fetch project statistics from GitLab. The script collects total storage size, repository size, job artifacts size, container registry size, wiki size, LFS usage, and the last activity timestamp. Exporting the results to a CSV gave us a clear overview and helped us pinpoint the main culprit: job artifacts. These are files produced by CI/CD jobs, such as build outputs, test reports, logs, or packaged applications, which can grow uncontrollably if not cleaned up regularly.

### Python Script to Collect GitLab Storage Data

    import requests
    import csv
    from datetime import datetime
    from zoneinfo import ZoneInfo
    
    GITLAB_URL = "https://git.company.com"
    PRIVATE_TOKEN = "adalahpokoknya"
    
    headers = {"PRIVATE-TOKEN": PRIVATE_TOKEN}
    all_projects = []
    
    for page in range(1, 11):
        r = requests.get(
            f"{GITLAB_URL}/api/v4/projects",
            headers=headers,
            params={
                "statistics": True,
                "order_by": "storage_size",
                "sort": "desc",
                "per_page": 100,
                "page": page
            }
        )
    
        if r.status_code != 200:
            print("Error:", r.text)
            break
    
        data = r.json()
        if not data:
            break
    
        all_projects.extend(data)
    
    with open("gitlab_storage_report.csv", mode="w", newline="", encoding="utf-8") as file:
        writer = csv.writer(file)
        writer.writerow([
            "Project", "Namespace", "Total_Storage_MB", "Repository_MB",
            "Artifacts_MB", "Registry_MB", "Wiki_MB", "LFS_MB",
            "Visibility", "Last_Activity"
        ])
    
        for p in all_projects:
            stats = p.get("statistics", {})
            last_activity_raw = p.get("last_activity_at")
            if last_activity_raw:
                dt = datetime.fromisoformat(last_activity_raw)
                dt_gmt7 = dt.astimezone(ZoneInfo("Asia/Jakarta"))
                last_activity = dt_gmt7.strftime("%Y-%m-%d %H:%M:%S")
            else:
                last_activity = ""
    
            writer.writerow([
                p.get("name"),
                p.get("path_with_namespace"),
                round(stats.get("storage_size", 0) / (1024 * 1024), 2),
                round(stats.get("repository_size", 0) / (1024 * 1024), 2),
                round(stats.get("job_artifacts_size", 0) / (1024 * 1024), 2),
                round(stats.get("container_registry_size", 0) / (1024 * 1024), 2),
                round(stats.get("wiki_size", 0) / (1024 * 1024), 2),
                round(stats.get("lfs_objects_size", 0) / (1024 * 1024), 2),
                p.get("visibility"),
                last_activity
            ])
    

After generating the CSV, it became evident that most storage was consumed by artifacts. Many projects were retaining artifacts indefinitely, and old pipelines were never cleaned. To reduce risk while reclaiming storage, we decided to keep the latest 70 percent of jobs and delete the oldest 30 percent. This approach protects recent pipelines while significantly reducing storage usage.

For automation, we leveraged Jenkins, which was already integrated in our CI/CD ecosystem. The cleanup pipeline runs daily and has two stages: a dry run and execution. The dry run fetches jobs, calculates total and deletable artifact storage, and generates a log and a deletion list without actually deleting anything. The execution stage deletes artifacts using the GitLab API.

### Jenkins Pipeline for Artifact Cleanup

    pipeline {
        agent {
            kubernetes {
                label 'cleanup-agent'
                defaultContainer 'cleanup'
                yaml """
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: cleanup
        image: alpine:3.23.3
        command:
        - cat
        tty: true
    """
            }
        }
    
        parameters {
            string(name: 'PROJECT_IDS_INPUT', defaultValue: '1938,5638,2568,2569,2566,2576,2570,5302,2007', description: 'Comma-separated Project IDs')
            booleanParam(name: 'RUN_DELETE', defaultValue: false, description: 'Set true to actually delete artifacts')
        }
    
        environment {
            LOG_FILE = 'artifact_cleanup.log'
            TOKEN = credentials('pat-gitlab-cleanup')
        }
    
        triggers {
            cron('H 3 * * *')
        }
    
        stages {
            stage('Cleanup GitLab Artifacts - Dry Run') {
                steps {
                    container('cleanup') {
                        sh '''
                        apk add --no-cache curl jq coreutils
                        echo "=== Artifact Cleanup Dry Run (Keep 70%) ===" > $LOG_FILE
                        : > delete_ids.txt
                        PROJECT_IDS=${PROJECT_IDS_INPUT:-'1938,5638,2568,2569,2566,2576,2570,5302,2007'}
                        ...
                        '''
                    }
                }
            }
    
            stage('Cleanup GitLab Artifacts - Execute') {
                when {
                    anyOf {
                        expression { return params.RUN_DELETE == true }
                        triggeredBy 'TimerTrigger'
                    }
                }
                steps {
                    container('cleanup') {
                        sh '''
                        apk add --no-cache curl
                        echo "=== Executing Artifact Cleanup ===" >> $LOG_FILE
                        ...
                        '''
                    }
                }
            }
    
            stage('Archive Logs') {
                steps {
                    container('cleanup') {
                        archiveArtifacts artifacts: 'artifact_cleanup.log', allowEmptyArchive: true
                    }
                }
            }
        }
    }
    

### Example Output

    Project 1938 - Total Jobs: 500
    Keep: 350
    Delete: 150
    
    Total Artifact Storage : 2048 MB
    To Be Deleted          : 820 MB
    Remaining After Cleanup: 1228 MB
    
    Global Summary:
    Projects processed       : 9
    Total Artifact Storage   : 10,240 MB
    Scheduled for Deletion   : 4,200 MB
    Remaining After Cleanup  : 6,040 MB
    

This pipeline allowed us to automatically clean up old artifacts, reclaiming storage without affecting active pipelines. By combining Python scripts for analysis, a clear retention policy, and Jenkins for automation, we were able to keep the GitLab instance manageable even without direct server access. The approach is safe, repeatable, and can be adapted for different retention policies per project or branch.

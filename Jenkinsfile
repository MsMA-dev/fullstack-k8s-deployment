import groovy.transform.Field

// Stage Tracker (Global)
@Field def STAGE_TRACKER = [
  map: [:],

  ok:   { name -> STAGE_TRACKER.map[name] = 'SUCCESS' },
  fail: { name -> STAGE_TRACKER.map[name] = 'FAILED' },

  toHtmlRows: {
    def order = [
      'Frontend Build & Test',
      'Backend Build & Test',
      'Sonar Frontend',
      'Sonar Backend',
      'Nexus Frontend',
      'Nexus Backend',
      'Docker Build & Push',
      'Deploy to Kubernetes'
    ]

    def esc = { s -> (s ?: '').replace('&','&amp;').replace('<','&lt;').replace('>','&gt;') }

    def badge = { st ->
      switch(st) {
        case 'SUCCESS':
          return "<span style='padding:4px 10px;border-radius:999px;background:#ecfdf3;color:#067647;font-weight:700;'>✓ SUCCESS</span>"
        case 'FAILED':
          return "<span style='padding:4px 10px;border-radius:999px;background:#fef2f2;color:#b42318;font-weight:700;'>✕ FAILED</span>"
        default:
          return "<span style='padding:4px 10px;border-radius:999px;background:#f3f4f6;color:#6b7280;font-weight:700;'>⊘ SKIPPED</span>"
      }
    }

    def html = ""
    order.eachWithIndex { n, i ->
      def st = STAGE_TRACKER.map.get(n, 'SKIPPED')
      def bg = (i % 2 == 0) ? "#ffffff" : "#fafafa"
      html += """
        <tr>
          <td style="padding:12px 14px;border-bottom:1px solid #f0f0f6;background:${bg};">${esc(n)}</td>
          <td style="padding:12px 14px;border-bottom:1px solid #f0f0f6;background:${bg};text-align:center;">${badge(st)}</td>
        </tr>
      """
    }
    return html
  }
]

pipeline {
    agent any
    environment {
        BACKEND_IMAGE   = "msmadev/fullstack-backend:${BUILD_NUMBER}"
        FRONTEND_IMAGE  = "msmadev/fullstack-frontend:${BUILD_NUMBER}"
    }

    stages {

        // Stage 1 --- CI: Build & Test (Frontend + Backend in parallel)
        stage('Build & Test') {
            parallel {
                stage('Frontend') {
                    steps {
                        script {
                            try {
                                dir('frontend') {
                                    sh 'npm ci'
                                    sh 'CHROME_BIN=/usr/bin/google-chrome xvfb-run -a npm test -- --watch=false --browsers=ChromeHeadless'
                                    sh 'npm run build -- --configuration production'
                                }
                                STAGE_TRACKER.ok('Frontend Build & Test')
                            } catch (e) {
                                STAGE_TRACKER.fail('Frontend Build & Test')
                                error("Frontend build/test failed: ${e.message}")
                            }
                        }
                    }
                }
                stage('Backend') {
                    steps {
                        script {
                            try {
                                dir('backend') { sh 'mvn clean install' }
                                STAGE_TRACKER.ok('Backend Build & Test')
                            } catch (e) {
                                STAGE_TRACKER.fail('Backend Build & Test')
                                error("Backend build failed: ${e.message}")
                            }
                        }
                    }
                }
            }
        }

        // Stage 2 --- SonarQube Analysis + Quality Gate (Frontend + Backend in parallel)
        stage('SonarQube') {
            parallel {
                stage('Sonar-Backend') {
                    steps {
                        script {
                            try {
                                dir('backend') {
                                    withSonarQubeEnv('SonarQube') {
                                        sh 'mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=fullstack-backend'
                                    }
                                    timeout(time: 15, unit: 'MINUTES') {
                                        def qg = waitForQualityGate abortPipeline: false
                                        if (qg.status != 'OK') error("Quality Gate failed: ${qg.status}")
                                    }
                                }
                                STAGE_TRACKER.ok('Sonar Backend')
                            } catch (e) {
                                STAGE_TRACKER.fail('Sonar Backend')
                                error("Backend SonarQube failed: ${e.message}")
                            }
                        }
                    }
                }
                stage('Sonar-Frontend') {
                    steps {
                        script {
                            try {
                                dir('frontend') {
                                    withSonarQubeEnv('SonarQube') {
                                        sh 'sonar-scanner -Dsonar.projectKey=fullstack-frontend -Dsonar.sources=src'
                                    }
                                    timeout(time: 15, unit: 'MINUTES') {
                                        def qg = waitForQualityGate abortPipeline: false
                                        if (qg.status != 'OK') error("Quality Gate failed: ${qg.status}")
                                    }
                                }
                                STAGE_TRACKER.ok('Sonar Frontend')
                            } catch (e) {
                                STAGE_TRACKER.fail('Sonar Frontend')
                                error("Frontend SonarQube failed: ${e.message}")
                            }
                        }
                    }
                }
            }
        }

        // Stage 3 --- Artifact Publishing to Nexus (Frontend + Backend in parallel)
        stage('Upload to Nexus') {
            parallel {
                stage('Backend - Nexus') {
                    steps {
                        script {
                            try {
                                dir('backend') {
                                    nexusArtifactUploader(
                                        artifacts: [[
                                          artifactId: 'demo', classifier: '', 
                                          file: 'target/demo-0.0.1-SNAPSHOT.jar', 
                                          type: 'jar'
                                          ]],
                                          credentialsId: 'Nexus',
                                          groupId: 'com.devops', nexusUrl: '51.44.31.9:8081',
                                          nexusVersion: 'nexus3', protocol: 'http',
                                          repository: 'fullstack-backend',
                                          version: "0.0.1-${BUILD_NUMBER}"
                                    )
                                }
                                STAGE_TRACKER.ok('Nexus Backend')
                            } catch (e) {
                                STAGE_TRACKER.fail('Nexus Backend')
                                error("Backend Nexus upload failed: ${e.message}")
                            }
                        }
                    }
                }
                stage('Frontend - Nexus') {
                    steps {
                        script {
                            try {
                                dir('frontend') {
                                    sh "tar -czf fullstack-frontend-${BUILD_NUMBER}.tgz -C dist ."
                                    nexusArtifactUploader(
                                        artifacts: [[
                                        artifactId: 'fullstack-frontend', classifier: '', 
                                        file: "fullstack-frontend-${BUILD_NUMBER}.tgz", type: 'tgz'
                                        ]],
                                        credentialsId: 'Nexus',
                                        groupId: 'com.devops', nexusUrl: '51.44.31.9:8081',
                                        nexusVersion: 'nexus3', protocol: 'http', 
                                        repository: 'fullstack-frontend',
                                        version: "0.0.1-${BUILD_NUMBER}"
                                    )
                                }
                                STAGE_TRACKER.ok('Nexus Frontend')
                            } catch (e) {
                                STAGE_TRACKER.fail('Nexus Frontend')
                                error("Frontend Nexus upload failed: ${e.message}")
                            }
                        }
                    }
                }
            }
        }

        // Stage 4 --- Containerization: Build & Push Docker Images to DockerHub
        stage('Docker Build & Push') {
            steps {
                script {
                    try {
                        withCredentials([
                          usernamePassword(
                          credentialsId: 'Docker',
                          usernameVariable: 'DOCKERHUB_USER', 
                          passwordVariable: 'DOCKERHUB_PASS')]) {
                            sh 'ansible-playbook ansible/build_push.yaml'
                        }
                        STAGE_TRACKER.ok('Docker Build & Push')
                    } catch (e) {
                        STAGE_TRACKER.fail('Docker Build & Push')
                        error("Docker build/push failed: ${e.message}")
                    }
                }
            }
        }

        // Stage 5 --- Deploy to Kubernetes Cluster via Ansible
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    try {
                        sh 'ansible-playbook ansible/deploy.yaml'
                        STAGE_TRACKER.ok('Deploy to Kubernetes')
                    } catch (e) {
                        STAGE_TRACKER.fail('Deploy to Kubernetes')
                        error("Kubernetes deployment failed: ${e.message}")
                    }
                }
            }
        }
    }

    // --- Post: always send email report regardless of build result
    post {
        always {
            script {
                def buildStatus   = currentBuild.currentResult ?: 'UNKNOWN'
                def buildDuration = currentBuild.durationString.replace(' and counting', '')
                def rowsHtml      = STAGE_TRACKER.toHtmlRows()

                def html = """<!DOCTYPE html>
                <html>
                <head>
                  <meta charset="utf-8"/>
                  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
                </head>
                <body style="font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Arial,sans-serif;background:#f6f7fb;margin:0;padding:24px;">
                  <div style="max-width:760px;margin:0 auto;background:#ffffff;border:1px solid #e8e8ee;border-radius:14px;overflow:hidden;box-shadow:0 4px 6px rgba(0,0,0,0.05);">

                    <div style="padding:20px 24px;background:linear-gradient(135deg,#1e293b 0%,#334155 100%);color:#ffffff;">
                      <div style="font-size:22px;font-weight:700;margin-bottom:8px;">Pipeline Report</div>
                      <div style="opacity:.9;font-size:14px;line-height:1.6;">
                        <div><strong>Job:</strong> ${env.JOB_NAME}</div>
                        <div><strong>Build:</strong> #${env.BUILD_NUMBER}</div>
                        <div><strong>Status:</strong>
                          <span style="background:${buildStatus == 'SUCCESS' ? '#10b981' : '#ef4444'};padding:2px 8px;border-radius:4px;">
                            ${buildStatus}
                          </span>
                        </div>
                        <div><strong>Duration:</strong> ${buildDuration}</div>
                      </div>
                    </div>

                    <div style="padding:24px;">
                      <h2 style="font-size:16px;color:#111827;margin:0 0 16px 0;font-weight:600;">Stage Summary</h2>

                      <table style="width:100%;border-collapse:separate;border-spacing:0;font-size:14px;border:1px solid #e5e7eb;border-radius:10px;overflow:hidden;">
                        <thead>
                          <tr>
                            <th style="text-align:left;padding:14px 16px;background:#f9fafb;font-weight:600;color:#374151;border-bottom:2px solid #e5e7eb;">Stage</th>
                            <th style="text-align:center;padding:14px 16px;background:#f9fafb;font-weight:600;color:#374151;border-bottom:2px solid #e5e7eb;">Status</th>
                          </tr>
                        </thead>
                        <tbody>
                          ${rowsHtml}
                        </tbody>
                      </table>

                      <div style="margin-top:20px;padding:16px;background:#f9fafb;border-radius:8px;">
                        <div style="font-size:13px;color:#6b7280;margin-bottom:12px;">
                          View complete build logs and console output in Jenkins.
                        </div>
                        <a href="${env.BUILD_URL}"
                           style="display:inline-block;padding:10px 20px;background:#2563eb;color:#ffffff;text-decoration:none;border-radius:8px;font-size:13px;font-weight:600;">
                          View Build →
                        </a>
                      </div>

                      <div style="margin-top:16px;font-size:12px;color:#9ca3af;padding-top:16px;border-top:1px solid #e5e7eb;">
                        <div>${new Date().format('yyyy-MM-dd HH:mm:ss')}</div>
                        <div>Jenkins URL: ${env.JENKINS_URL}</div>
                      </div>
                    </div>

                  </div>
                </body>
                </html>"""

                emailext(
                    to: 'm.alotaibi.cs@gmail.com',
                    subject: "Jenkins Build ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    body: html,
                    attachLog: buildStatus == 'FAILURE'   // attach log only on failure
                )
            }
        }
    }
}

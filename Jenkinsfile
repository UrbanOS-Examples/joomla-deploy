library(
    identifier: 'pipeline-lib@4.8.0',
    retriever: modernSCM([$class: 'GitSCMSource',
                          remote: 'https://github.com/SmartColumbusOS/pipeline-lib',
                          credentialsId: 'jenkins-github-user'])
)

properties([
    pipelineTriggers([scos.dailyBuildTrigger()]),
    parameters([
        booleanParam(defaultValue: false, description: 'Deploy to development environment?', name: 'DEV_DEPLOYMENT'),
        string(defaultValue: 'development', description: 'Image tag to deploy to dev environment', name: 'DEV_IMAGE_TAG')
    ])
])

def image
def doStageIf = scos.&doStageIf
def doStageIfDeployingToDev = doStageIf.curry(env.DEV_DEPLOYMENT == "true")
def doStageIfMergedToMaster = doStageIf.curry(scos.changeset.isMaster && env.DEV_DEPLOYMENT == "false")
def doStageIfRelease = doStageIf.curry(scos.changeset.isRelease)

node ('infrastructure') {
    ansiColor('xterm') {
        scos.doCheckoutStage()


        doStageIfDeployingToDev('Deploy to Dev') {
            deployTo(environment: 'dev', extraArgs: "--set image.tag=${env.DEV_IMAGE_TAG} --set image.pullPolicy=Always --recreate-pods")
        }

        doStageIfMergedToMaster('Deploy to Staging')  {
            deployTo(environment: 'staging')
            scos.applyAndPushGitHubTag('staging')
        }

        doStageIfRelease('Deploy to Production') {
            deployTo(environment: 'prod', internal: false)
            scos.applyAndPushGitHubTag('prod')
        }
    }
}

def deployTo(params = [:]) {
    def environment = params.get('environment')
    if (environment == null) throw new IllegalArgumentException("environment must be specified")

    def internal = params.get('internal', true)
    def extraArgs = params.get('extraArgs', '')

    scos.withEksCredentials(environment) {

        def terraformOutputs = scos.terraformOutput(environment)
        def subnets = terraformOutputs.public_subnets.value.join(/\\,/)
        def allowInboundTrafficSG = terraformOutputs.allow_all_security_group.value
        def rootIngressScheme = internal ? 'internal' : 'internet-facing'
        def rootCertificateARN = terraformOutputs.root_tls_certificate_arn.value
        def internalCertificateARN = terraformOutputs.tls_certificate_arn.value
        def rootDnsZone = terraformOutputs.root_dns_zone_name.value
        def internalDnsZone = terraformOutputs.internal_dns_zone_name.value
        def rootWAFACLARN = terraformOutputs.eks_cluster_waf_acl_arn.value

        sh("""#!/bin/bash
            set -e
            helm init --client-only
            helm upgrade --install scos-joomla ./chart \
                --namespace=joomla \
                --values=joomla.yaml \
                --set ingress.subnets="${subnets}" \
                --set ingress.security_groups="${allowInboundTrafficSG}" \
                --set ingress.root.enabled="true" \
                --set ingress.root.scheme="${rootIngressScheme}" \
                --set ingress.root.dns_zone="${rootDnsZone}" \
                --set ingress.root.certificate_arns="${rootCertificateARN}" \
                --set ingress.root.waf_acl_arn="${rootWAFACLARN}" \
                --set ingress.internal.enabled="true" \
                --set ingress.internal.dns_zone="${internalDnsZone}" \
                --set ingress.internal.certificate_arns="${internalCertificateARN}" \
                ${extraArgs}
        """.trim())
    }
}

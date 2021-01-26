@Library('pmd@family-pmd4') _

import uk.org.floop.jenkins_pmd.Drafter
import uk.org.floop.jenkins_pmd.PMDConfig
import uk.org.floop.jenkins_pmd.models.CatalogMetadata

pipeline {
    agent {
        label 'master'
    }
    stages {
        stage('Add referenced vocabs') {
            agent {
                docker {
                    image 'gsscogs/csv2rdf'
                    reuseNode true
                    alwaysPull true
                }
            }
            steps {
                script {
                    def pmd = pmdConfig('pmd')
                    for (myDraft in pmd.drafter
                            .listDraftsets(Drafter.Include.OWNED)
                            .findAll { it['display-name'] == env.JOB_NAME }) {
                        pmd.drafter.deleteDraftset(myDraft.id)
                    }
                    def id = pmd.drafter.createDraftset(env.JOB_NAME).id
                    for (graph in util.jobGraphs(pmd, id)) {
                        pmd.drafter.deleteGraph(id, graph)
                        echo "Removing own graph ${graph}"
                    }

                    for (vocab in readJSON(file: 'vocabs/index.json')) {
                        String src = vocab.src
                        def fileFriendlyIdentifier = src.replaceAll(/[^A-Za-z0-9]+/, "-")
                        echo "Adding ${src} (${fileFriendlyIdentifier})..."

                        String graph, localFilePath
                        if (vocab.src.startsWith('http')) {
                            graph = vocab.src

                            def fileContents = util.getUrlAsText(vocab.src, vocab.format)
                            String downloadedFileExtension
                            switch (vocab.format.toLowerCase()) {
                                case 'application/rdf+xml':
                                    downloadedFileExtension = 'rdf'
                                    break
                                case 'application/n-triples':
                                    downloadedFileExtension = 'nt'
                                    break
                                case 'text/turtle':
                                    downloadedFileExtension = 'ttl'
                                    break
                                default:
                                    throw new Exception("Download file type '${vocab.format}' has not been configured.")
                            }

                            def downloadedFilePath = "${WORKSPACE}/${fileFriendlyIdentifier}.${downloadedFileExtension}"
                            writeFile(file: downloadedFilePath, text: fileContents)
                            localFilePath = downloadedFilePath
                        } else {
                            graph = vocab.graph
                            localFilePath = "${WORKSPACE}/${vocab.src}"
                        }
                        // Standardise the format so we can augment it if necessary
                        def standardisedFormatOutputFilePath = "${WORKSPACE}/${fileFriendlyIdentifier}.standard.out.ttl"
                        sh "sparql --data \"${localFilePath}\" 'CONSTRUCT {?s ?p ?o.} WHERE {?s ?p ?o.}' > \"${standardisedFormatOutputFilePath}\""
                        
                        sh "sparql --data \"${localFilePath}\" 'SELECT (COUNT(*) as ?numTriples) WHERE { ?s ?p ?o. }'"

                        if (vocab.filter != null) {
                            for (filterQueryFilePath in vocab.filter) {
                                echo "Filtering with ${filterQueryFilePath}"
                                // N.B. **Overwrites** standardised.format.ttl with filtered data.
                                sh "sparql --data \"${standardisedFormatOutputFilePath}\" --query \"${WORKSPACE}/${filterQueryFilePath}\" > \"${standardisedFormatOutputFilePath}\""
                                sh "sparql --data \"${localFilePath}\" 'SELECT (COUNT(*) as ?numTriples) WHERE { ?s ?p ?o. }'"
                            }
                        }

                        if (vocab.augment != null) {
                            for (augmentationQueryFilePath in vocab.augment) {
                                echo "Augmenting with ${augmentationQueryFilePath}"
                                sh "sparql --data \"${standardisedFormatOutputFilePath}\" --query \"${WORKSPACE}/${augmentationQueryFilePath}\" >> \"${standardisedFormatOutputFilePath}\""
                                sh "sparql --data \"${localFilePath}\" 'SELECT (COUNT(*) as ?numTriples) WHERE { ?s ?p ?o. }'"
                            }
                        }

                        pmd.drafter.deleteGraph(id, graph)
                        pmd.drafter.addData(id, standardisedFormatOutputFilePath, "text/turtle", "UTF-8", graph)

                        if (vocab.conceptSchemes != null){
                            for (conceptScheme in vocab.conceptSchemes) {
                                def catalogMetadata = new CatalogMetadata(conceptScheme)
                                def metadataFilePath = "${WORKSPACE}/${fileFriendlyIdentifier}.${catalogMetadata.identifier}.catalog.out.ttl"
                                writeFile(file: metadataFilePath, text: util.getCatalogMetadata(graph, catalogMetadata))
                                pmd.drafter.addData(id, metadataFilePath, "text/turtle", 'UTF-8', graph)
                            }
                        }

                        writeFile(file: "prov.ttl", text: util.jobPROV(graph))
                        pmd.drafter.addData(id, "${WORKSPACE}/prov.ttl", "text/turtle", "UTF-8", graph)
                    }
                    pmd.drafter.publishDraftset(id)
                }
            }
        }
    }
    post {
            always {
                script {
                    archiveArtifacts artifacts: "**/*.out.ttl"
                }
            }
    }
}

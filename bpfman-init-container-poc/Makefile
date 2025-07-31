
TAG ?= latest
NAMESPACE ?= bpftest

.PHONY: build
build:
	podman build -t quay.io/akaris/bpfman-init-container:$(TAG) -f ../Containerfile.bpfman.initcontainer ../.

.PHONY: push
push:
	podman push quay.io/akaris/bpfman-init-container:$(TAG)

.PHONY: deploy-namespace
deploy-namespace: 
	kubectl create namespace $(NAMESPACE)
	kubectl config set-context --current --namespace=bpftest
	kubectl label ns $(NAMESPACE) security.openshift.io/scc.podSecurityLabelSync="false" --overwrite=true
	kubectl label ns $(NAMESPACE) pod-security.kubernetes.io/enforce=privileged --overwrite=true
	kubectl label ns $(NAMESPACE) pod-security.kubernetes.io/warn=privileged --overwrite=true
	kubectl label ns $(NAMESPACE) pod-security.kubernetes.io/audit=privileged --overwrite=true

.PHONY: deploy
deploy: deploy-namespace
	kubectl apply -f bpfman-init-container-test.yaml

.PHONY: undeploy
undeploy:
	kubectl delete -f bpfman-init-container-test.yaml || true
	kubectl config set-context --current --namespace=default
	kubectl delete namespace $(NAMESPACE) || true

.PHONY: logs
logs:
	kubectl get pods bpfman-init-container-test	
	@echo -e "\nbpfman-init\n================"
	kubectl logs -c bpfman-init bpfman-init-container-test
	@echo -e "\nmain-container\n================"
	kubectl logs -c main-container bpfman-init-container-test

# Variables
VAULT_IMAGE ?= hashicorp/vault:latest
SECUREVAULT_IMAGE ?= europe-docker.pkg.dev/jenkins-189019/igrantio/securevault:2025.2.4
VAULT_CONTAINER_NAME := vault

# Detect platform
UNAME_S := $(shell uname -s)
ifeq ($(UNAME_S),Linux)
    DOCKER_EXEC_FLAGS := -it
else
    DOCKER_EXEC_FLAGS := -it
endif

.PHONY: help

# Show help - automatically generated from comments
.DEFAULT_GOAL := help
.PHONY: help
help:
	@echo "------------------------------------------------------------------------"
	@echo "iGrant.io Platform"
	@echo "------------------------------------------------------------------------"
	@grep -E '^[0-9a-zA-Z_/%\-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

vault-start: ## Start Vault server in detached mode
	@echo "Starting Vault server..."
	@docker compose up -d vault
	@echo "Vault server started successfully"

vault-init: ## Initialize Vault and generate root token
	@echo "Checking if Vault is already initialized..."
	@if [ ! -f resources/config/cluster-keys.json ]; then \
		echo "Initializing Vault..."; \
		docker exec $(DOCKER_EXEC_FLAGS) $(VAULT_CONTAINER_NAME) vault operator init \
			-key-shares=1 \
			-key-threshold=1 \
			-format=json > resources/config/cluster-keys.json; \
		echo "Vault initialized successfully"; \
	else \
		echo "Vault is already initialized. Using existing keys."; \
	fi

vault-clean: ## Delete Vault cluster keys file
	@echo "Cleaning Vault cluster keys..."
	@if [ -f resources/config/cluster-keys.json ]; then \
		rm resources/config/cluster-keys.json; \
		echo "Cluster keys file deleted successfully"; \
	else \
		echo "No cluster keys file found"; \
	fi

vault-status: ## Check current Vault server status
	@echo "Checking Vault status..."
	@docker exec $(DOCKER_EXEC_FLAGS) $(VAULT_CONTAINER_NAME) vault status
	@echo "Status check completed"

vault-unseal: ## Unseal Vault using stored key
	@echo "Unsealing Vault..."
	@docker exec $(DOCKER_EXEC_FLAGS) $(VAULT_CONTAINER_NAME) vault operator unseal $$(jq -r '.unseal_keys_b64[]' resources/config/cluster-keys.json)
	@echo "Vault unsealed successfully"

vault-token: ## Print root token and copy to clipboard
	@echo "Root token:"
	@jq -r '.root_token' resources/config/cluster-keys.json | tee /dev/stderr | \
	{ read token; \
		if command -v pbcopy > /dev/null 2>&1; then \
			echo "$$token" | pbcopy && echo "Token copied to clipboard!"; \
		elif command -v xclip > /dev/null 2>&1; then \
			echo "$$token" | xclip -selection clipboard && echo "Token copied to clipboard!"; \
		elif command -v clip.exe > /dev/null 2>&1; then \
			echo "$$token" | clip.exe && echo "Token copied to clipboard!"; \
		else \
			echo "No clipboard command found. Please copy the token manually."; \
		fi; \
	}

vault-configure: ## Configure Vault with basic configuration
	@chmod +x resources/scripts/configure-vault.sh
	@./resources/scripts/configure-vault.sh

vault-bootstrap: ## Start Vault, initialize, unseal and configure it
	@echo "Starting Vault bootstrap process..."
	@docker rm -f $(VAULT_CONTAINER_NAME) 2>/dev/null || true
	@make vault-start
	@echo "Waiting for Vault to start..."
	@sleep 5
	@max_attempts=30; \
	attempt=1; \
	until docker exec $(VAULT_CONTAINER_NAME) vault status 2>/dev/null | grep -q "Sealed" || [ $$attempt -gt $$max_attempts ]; do \
		echo "Waiting for Vault to be ready... (Attempt $$attempt/$$max_attempts)"; \
		sleep 2; \
		attempt=$$((attempt + 1)); \
	done; \
	if [ $$attempt -gt $$max_attempts ]; then \
		echo "Error: Vault failed to start after $$max_attempts attempts"; \
		exit 1; \
	fi
	@make vault-init
	@make vault-unseal
	@make vault-configure
	@echo "Vault bootstrap completed successfully!"

securevault-start: ## Start iGrant.io secure vault
	@echo "Starting Vault server..."
	@docker compose up -d securevault
	@echo "Vault server started successfully"

mongo-start: ## Start MongoDB server
	@echo "Starting MongoDB server..."
	@docker compose up -d mongo
	@echo "MongoDB server started successfully"

postgresql-start: ## Start PostgreSQL server
	@echo "Starting PostgreSQL server..."
	@docker compose up -d postgresql
	@echo "PostgreSQL server started successfully"

keycloak-start: ## Start Keycloak server
	@echo "Waiting for PostgreSQL to start..."
	@max_attempts=30; \
	attempt=1; \
	until docker logs $$(docker compose ps -q postgresql) 2>&1 | grep -q "database system is ready to accept connections" || [ $$attempt -gt $$max_attempts ]; do \
		echo "Waiting for PostgreSQL to be ready... (Attempt $$attempt/$$max_attempts)"; \
		sleep 2; \
		attempt=$$((attempt + 1)); \
	done; \
	if [ $$attempt -gt $$max_attempts ]; then \
		echo "Error: PostgreSQL failed to start after $$max_attempts attempts"; \
		exit 1; \
	fi
	@echo "PostgreSQL is ready. Starting Keycloak..."
	@docker compose up -d keycloak
	@echo "Keycloak server started successfully"

api-start: ## Start API server
	@echo "Starting API server..."
	@docker compose up -d api
	@echo "API server started successfully"

webhook-start: ## Start Webhook server
	@echo "Starting Webhook server..."
	@docker compose up -d webhook
	@echo "Webhook server started successfully"

owconfig-start: ## Start Organisation Wallet Config server
	@echo "Starting Organisation Wallet Config server..."
	@docker compose up -d owconfig
	@echo "Organisation Wallet Config server started successfully"

owservice-start: ## Start Organisation Wallet Service server
	@echo "Starting Organisation Wallet Service server..."
	@docker compose up -d owservice
	@echo "Organisation Wallet Service server started successfully"

configure-json: ## Update config-production.json with environment variables
	@echo "Updating config-production.json with environment variables..."
	@cp resources/config/config-production.json resources/config/config-production.json.bak
ifeq ($(UNAME_S),Linux)
	@jq \
		--arg db_name "$$MONGODB_DATABASE" \
		--arg db_user "$$MONGODB_USER" \
		--arg db_pass "$$MONGODB_PASSWORD" \
		--arg kc_user "$$KEYCLOAK_USER" \
		--arg kc_pass "$$KEYCLOAK_PASSWORD" \
		--arg openidserverbaseurl "http://owconfig:8080" \
		--arg apipublicbaseurl "$$API_PUBLIC_BASE_URL" \
		--arg owservicepublicbaseurl "$$OWSERVICE_PUBLIC_BASE_URL" \
		--arg kc_url "http://$$KEYCLOAK_CONTAINER_NAME:8080/auth" \
		'.DataBase.name = $$db_name | .DataBase.username = $$db_user | .DataBase.password = $$db_pass | .Iam.url = $$kc_url | .Iam.AdminUser = $$kc_user | .Iam.AdminPassword = $$kc_pass | .SSIAriesCloudAgentDeployment.BackendAPIBaseURL = $$apipublicbaseurl | .OpenIdDeployment.OpenIdServiceEndpoint = $$owservicepublicbaseurl | .OpenIdDeployment.OpenIdServerBaseUrl = $$openidserverbaseurl' \
		resources/config/config-production.json.bak > resources/config/config-production.json
else
	@jq --arg db_name "$$MONGODB_DATABASE" \
		--arg db_user "$$MONGODB_USER" \
		--arg db_pass "$$MONGODB_PASSWORD" \
		--arg kc_user "$$KEYCLOAK_USER" \
		--arg kc_pass "$$KEYCLOAK_PASSWORD" \
		--arg openidserverbaseurl "http://owconfig:8080" \
		--arg apipublicbaseurl "$$API_PUBLIC_BASE_URL" \
		--arg owservicepublicbaseurl "$$OWSERVICE_PUBLIC_BASE_URL" \
		--arg kc_url "http://$$KEYCLOAK_CONTAINER_NAME:8080/auth" \
		'.DataBase.name = $$db_name | \
		.DataBase.username = $$db_user | \
		.DataBase.password = $$db_pass | \
		.Iam.url = $$kc_url | \
		.Iam.AdminUser = $$kc_user | \
		.Iam.AdminPassword = $$kc_pass | \
		.SSIAriesCloudAgentDeployment.BackendAPIBaseURL = $$apipublicbaseurl | \
		.OpenIdDeployment.OpenIdServiceEndpoint = $$owservicepublicbaseurl | \
		.OpenIdDeployment.OpenIdServerBaseUrl = $$openidserverbaseurl' \
		resources/config/config-production.json.bak > resources/config/config-production.json
endif
	@rm resources/config/config-production.json.bak
	@echo "Configuration file updated successfully!"

portainer-start: ## Start Portainer container management UI
	@echo "Starting Portainer server..."
	@docker compose -f docker-compose-portainer.yaml up -d portainer
	@echo "Portainer server started successfully"

# Stop routines for individual services
vault-stop: ## Stop Vault server
	@echo "Stopping Vault server..."
	@docker compose stop vault
	@echo "Vault server stopped successfully"

securevault-stop: ## Stop SecureVault server
	@echo "Stopping SecureVault server..."
	@docker compose stop securevault
	@echo "SecureVault server stopped successfully"

mongo-stop: ## Stop MongoDB server
	@echo "Stopping MongoDB server..."
	@docker compose stop mongo
	@echo "MongoDB server stopped successfully"

postgresql-stop: ## Stop PostgreSQL server
	@echo "Stopping PostgreSQL server..."
	@docker compose stop postgresql
	@echo "PostgreSQL server stopped successfully"

keycloak-stop: ## Stop Keycloak server
	@echo "Stopping Keycloak server..."
	@docker compose stop keycloak
	@echo "Keycloak server stopped successfully"

api-stop: ## Stop API server
	@echo "Stopping API server..."
	@docker compose stop api
	@echo "API server stopped successfully"

webhook-stop: ## Stop Webhook server
	@echo "Stopping Webhook server..."
	@docker compose stop webhook
	@echo "Webhook server stopped successfully"

owconfig-stop: ## Stop Organisation Wallet Config server
	@echo "Stopping Organisation Wallet Config server..."
	@docker compose stop owconfig
	@echo "Organisation Wallet Config server stopped successfully"

owservice-stop: ## Stop Organisation Wallet Service server
	@echo "Stopping Organisation Wallet Service server..."
	@docker compose stop owservice
	@echo "Organisation Wallet Service server stopped successfully"

# Volume cleanup routines
clean-volumes: ## Clean up Docker volumes (Usage: make clean-volumes VOLUMES=yes to confirm deletion)
	@if [ "$(VOLUMES)" = "yes" ]; then \
		echo "Cleaning up Docker volumes..."; \
		docker compose down -v; \
		echo "Volumes cleaned up successfully"; \
	else \
		echo "To clean up volumes, run: make clean-volumes VOLUMES=yes"; \
		echo "WARNING: This will permanently delete all data in the volumes!"; \
	fi

stop-all: ## Stop all containers
	@echo "Stopping all containers..."
	@docker compose stop
	@echo "All containers stopped successfully"

down: ## Stop and remove all containers
	@echo "Stopping and removing all containers..."
	@docker compose down
	@echo "All containers stopped and removed successfully"

start-all: ## Start all services in the correct order with proper initialization and safety delays
	@echo "Starting all services..."
	@if [ ! -f env.sh ]; then \
		echo "Error: env.sh file not found!"; \
		echo "Please ensure env.sh exists with required environment variables."; \
		exit 1; \
	fi
	@/bin/bash -c '\
		echo "Loading environment variables..." && \
		source ./env.sh && \
		echo "Configuring JSON..." && \
		$(MAKE) configure-json && \
		echo "Waiting 5s..." && sleep 5 && \
		echo "Bootstrapping Vault..." && \
		if ! $(MAKE) vault-bootstrap 2>&1 | tee /dev/stderr | grep -q "Vault is not initialized"; then \
			echo "Vault bootstrap successful."; \
		else \
			echo "Detected Vault initialization error. Cleaning up..." && \
			docker volume rm vault-data 2>/dev/null || true && \
			$(MAKE) vault-clean && \
			echo "Retrying Vault bootstrap..." && \
			$(MAKE) vault-bootstrap; \
		fi && \
		echo "Waiting 10s..." && sleep 10 && \
		echo "Starting SecureVault..." && \
		$(MAKE) securevault-start && \
		echo "Waiting 10s..." && sleep 10 && \
		echo "Starting MongoDB..." && \
		$(MAKE) mongo-start && \
		echo "Waiting 15s..." && sleep 15 && \
		echo "Starting PostgreSQL..." && \
		$(MAKE) postgresql-start && \
		echo "Waiting 15s..." && sleep 15 && \
		echo "Starting Keycloak..." && \
		$(MAKE) keycloak-start && \
		echo "Waiting 20s..." && sleep 20 && \
		echo "Starting API..." && \
		$(MAKE) api-start && \
		echo "Waiting 10s..." && sleep 10 && \
		echo "Starting Webhook..." && \
		$(MAKE) webhook-start && \
		echo "Waiting 10s..." && sleep 10 && \
		echo "Starting Organisation Wallet Config..." && \
		$(MAKE) owconfig-start && \
		echo "Waiting 10s..." && sleep 10 && \
		echo "Starting Organisation Wallet Service..." && \
		$(MAKE) owservice-start && \
		echo "Waiting 5s..." && sleep 5 && \
		echo "All services started successfully!"'

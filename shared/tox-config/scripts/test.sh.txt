#!/usr/bin/env bash
# ==============================================================================
# test.bash - Execution sequentielle des tests multi-environnements
# ==============================================================================
#
# Ce script lit le fichier .tox-config/versions.txt et execute les tests
# pour chaque version Python de maniere sequentielle.
# En cas d'echec, le script s'arrete immediatement.
#
# Usage:
#   ./.tox-config/scripts/test.bash [versions_file]
#
# Arguments:
#   versions_file: Chemin vers le fichier de versions 
#                  (defaut: .tox-config/versions.txt)
#
# Codes de sortie:
#   0: Tous les tests ont reussi
#   1: Au moins un test a echoue
#   2: Fichier de versions introuvable
#
# ==============================================================================

set -euo pipefail  # Arret immediat en cas d'erreur

# Couleurs pour l'affichage
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly BLUE='\033[0;34m'
readonly NC='\033[0m' # No Color

# Configuration
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly PROJECT_ROOT="$(cd "${SCRIPT_DIR}/../.." && pwd)"
readonly VERSIONS_FILE="${1:-${PROJECT_ROOT}/.tox-config/versions.txt}"

# ==============================================================================
# Fonctions utilitaires
# ==============================================================================

log_info() {
    echo -e "${BLUE}[INFO]${NC} $*"
}

log_success() {
    echo -e "${GREEN}[SUCCESS]${NC} $*"
}

log_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $*"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $*" >&2
}

log_section() {
    echo ""
    echo -e "${BLUE}================================================================${NC}"
    echo -e "${BLUE}$*${NC}"
    echo -e "${BLUE}================================================================${NC}"
    echo ""
}

# ==============================================================================
# Fonction principale
# ==============================================================================

run_tests() {
    cd "${PROJECT_ROOT}"
    
    # Verification de l'existence du fichier
    if [[ ! -f "${VERSIONS_FILE}" ]]; then
        log_error "File not found: ${VERSIONS_FILE}"
        log_info "Using default versions: py310, py311"
        
        # Fallback sur versions par defaut
        local default_versions=("py310" "py311")
        for env in "${default_versions[@]}"; do
            log_section "Running tests with ${env}"
            if ! tox -e "${env}"; then
                log_error "Tests failed for ${env}"
                return 1
            fi
            log_success "Tests passed for ${env}"
        done
        return 0
    fi
    
    log_info "Reading test versions from: ${VERSIONS_FILE}"
    
    # Lecture et traitement du fichier
    local versions=()
    while IFS= read -r line || [[ -n "${line}" ]]; do
        # Suppression des espaces en debut et fin (sans xargs)
        line="${line#"${line%%[![:space:]]*}"}"  # Trim leading
        line="${line%"${line##*[![:space:]]}"}"  # Trim trailing
        
        # Ignorer les lignes vides et les commentaires
        [[ -z "${line}" ]] && continue
        [[ "${line}" =~ ^# ]] && continue
        
        versions+=("${line}")
    done < "${VERSIONS_FILE}"
    
    # Verification qu'au moins une version est definie
    if [[ ${#versions[@]} -eq 0 ]]; then
        log_warning "No test versions found in ${VERSIONS_FILE}"
        log_info "Using default versions: py310, py311"
        versions=("py310" "py311")
    fi
    
    log_info "Test versions: ${versions[*]}"
    echo ""
    
    # Execution sequentielle des tests
    local total=${#versions[@]}
    local current=0
    
    for env in "${versions[@]}"; do
        ((current++))
        log_section "Running tests with ${env} (${current}/${total})"
        
        if ! tox -e "${env}"; then
            log_error "Tests failed for ${env}"
            log_error "Stopping test execution (${current}/${total} completed)"
            return 1
        fi
        
        log_success "Tests passed for ${env} (${current}/${total})"
    done
    
    echo ""
    log_success "All tests passed! (${total}/${total})"
    return 0
}

# ==============================================================================
# Point d'entree
# ==============================================================================

main() {
    log_info "Starting sequential test execution"
    log_info "Working directory: ${PROJECT_ROOT}"
    
    if run_tests; then
        log_success "Test execution completed successfully"
        exit 0
    else
        log_error "Test execution failed"
        exit 1
    fi
}

# Execution
main "$@"
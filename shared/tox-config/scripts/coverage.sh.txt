#!/usr/bin/env bash
# ==============================================================================
# coverage.bash - Generation du rapport de couverture de code
# ==============================================================================
#
# Ce script lit le fichier .tox-config/coverage-version.txt pour determiner
# quelle version Python utiliser pour generer le rapport de couverture.
#
# Usage:
#   ./.tox-config/scripts/coverage.bash [coverage_version_file]
#
# Arguments:
#   coverage_version_file: Chemin vers le fichier de version coverage
#                          (defaut: .tox-config/coverage-version.txt)
#
# Codes de sortie:
#   0: Rapport genere avec succes
#   1: Echec de generation du rapport
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
readonly COVERAGE_VERSION_FILE="${1:-${PROJECT_ROOT}/.tox-config/coverage-version.txt}"
readonly DEFAULT_COVERAGE_ENV="py312"

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
# Lecture de la version coverage
# ==============================================================================

get_coverage_env() {
    local coverage_env="${DEFAULT_COVERAGE_ENV}"
    
    if [[ ! -f "${COVERAGE_VERSION_FILE}" ]]; then
        log_warning "File not found: ${COVERAGE_VERSION_FILE}"
        log_info "Using default coverage environment: ${DEFAULT_COVERAGE_ENV}"
        echo "${coverage_env}"
        return 0
    fi
    
    # Lecture de la premiere ligne non vide et non commentaire
    while IFS= read -r line || [[ -n "${line}" ]]; do
        # Suppression des espaces en debut et fin (sans xargs)
        line="${line#"${line%%[![:space:]]*}"}"  # Trim leading
        line="${line%"${line##*[![:space:]]}"}"  # Trim trailing
        
        # Ignorer lignes vides et commentaires
        [[ -z "${line}" ]] && continue
        [[ "${line}" =~ ^# ]] && continue
        
        # Premiere ligne valide = version coverage
        coverage_env="${line}"
        break
    done < "${COVERAGE_VERSION_FILE}"
    
    echo "${coverage_env}"
}

# ==============================================================================
# Generation du rapport
# ==============================================================================

generate_coverage() {
    cd "${PROJECT_ROOT}"
    
    local coverage_env
    coverage_env=$(get_coverage_env)
    
    log_info "Coverage environment: ${coverage_env}"
    log_section "Generating coverage report with ${coverage_env}"
    
    # Utiliser tox -e coverage si c'est py312 (environnement dedie)
    # Sinon utiliser l'environnement specifie
    if [[ "${coverage_env}" == "py312" ]] || [[ "${coverage_env}" == "default" ]]; then
        log_info "Using dedicated coverage environment"
        if ! tox -e coverage; then
            log_error "Coverage generation failed"
            return 1
        fi
    else
        log_info "Using ${coverage_env} environment for coverage"
        
        # Creer le repertoire coverage s'il n'existe pas
        mkdir -p coverage
        
        # Executer pytest avec coverage dans l'environnement specifie
        if ! tox -e "${coverage_env}" -- \
            pytest --import-mode=importlib \
            --cov=ndict_tools \
            --cov-report=term-missing \
            --cov-report=xml:coverage/coverage.xml \
            --cov-report=html:coverage/coverage_html tests; then
            log_error "Coverage generation failed with ${coverage_env}"
            return 1
        fi
    fi
    
    log_success "Coverage report generated successfully"
    
    # Afficher les chemins des rapports
    echo ""
    log_info "Coverage reports:"
    [[ -f "coverage/coverage.xml" ]] && log_info "  - XML: coverage/coverage.xml"
    [[ -d "coverage/coverage_html" ]] && log_info "  - HTML: coverage/coverage_html/index.html"
    
    return 0
}

# ==============================================================================
# Point d'entree
# ==============================================================================

main() {
    log_info "Starting coverage report generation"
    log_info "Working directory: ${PROJECT_ROOT}"
    log_info "Coverage version file: ${COVERAGE_VERSION_FILE}"
    
    if generate_coverage; then
        log_success "Coverage generation completed successfully"
        exit 0
    else
        log_error "Coverage generation failed"
        exit 1
    fi
}

# Execution
main "$@"
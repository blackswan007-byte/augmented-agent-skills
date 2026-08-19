name: Create Release

on:  
  push:  
    branches:  
      \- main  
    paths:  
      \- 'pyproject.toml'  
  workflow\_dispatch:

permissions:  
  contents: write

jobs:  
  release:  
    runs-on: ubuntu-latest  
      
    steps:  
      \- name: Checkout repository  
        uses: actions/checkout@v6  
        with:  
          fetch-depth: 0  \# Fetch all history for release notes  
        
      \- name: Extract version from pyproject.toml  
        id: get\_version  
        run: |  
          VERSION=$(grep '^version' pyproject.toml | head \-1 | sed 's/.\*"\\(.\*\\)".\*/\\1/')  
          echo "version=$VERSION" \>\> $GITHUB\_OUTPUT  
          echo "tag=v$VERSION" \>\> $GITHUB\_OUTPUT  
          echo "Extracted version: $VERSION"  
        
      \- name: Check if tag already exists  
        id: check\_tag  
        run: |  
          if git rev-parse "v${{ steps.get\_version.outputs.version }}" \>/dev/null 2\>&1; then  
            echo "exists=true" \>\> $GITHUB\_OUTPUT  
            echo "Tag v${{ steps.get\_version.outputs.version }} already exists"  
          else  
            echo "exists=false" \>\> $GITHUB\_OUTPUT  
            echo "Tag v${{ steps.get\_version.outputs.version }} does not exist"  
          fi  
        
      \- name: Get previous tag  
        id: previous\_tag  
        if: steps.check\_tag.outputs.exists \== 'false'  
        run: |  
          PREVIOUS\_TAG=$(git describe \--tags \--abbrev=0 2\>/dev/null || echo "")  
          if \[ \-z "$PREVIOUS\_TAG" \]; then  
            echo "previous\_tag=" \>\> $GITHUB\_OUTPUT  
            echo "No previous tag found"  
          else  
            echo "previous\_tag=$PREVIOUS\_TAG" \>\> $GITHUB\_OUTPUT  
            echo "Previous tag: $PREVIOUS\_TAG"  
          fi  
        
      \- name: Generate release notes  
        id: release\_notes  
        if: steps.check\_tag.outputs.exists \== 'false'  
        run: |  
          PREVIOUS\_TAG="${{ steps.previous\_tag.outputs.previous\_tag }}"  
            
          \# Start release notes  
          cat \> release\_notes.md \<\< 'EOF'  
          \#\# What's Changed  
            
          EOF  
            
          \# Generate changelog from commits  
          if \[ \-n "$PREVIOUS\_TAG" \]; then  
            echo "Changes since $PREVIOUS\_TAG:" \>\> release\_notes.md  
            echo "" \>\> release\_notes.md  
              
            \# Get commits with nice formatting  
            git log ${PREVIOUS\_TAG}..HEAD \--pretty=format:"\* %s (%h)" \--no-merges \>\> release\_notes.md  
          else  
            echo "Initial release of Claude Scientific Skills" \>\> release\_notes.md  
            echo "" \>\> release\_notes.md  
            echo "This release includes:" \>\> release\_notes.md  
            git log \--pretty=format:"\* %s (%h)" \--no-merges \--max-count=20 \>\> release\_notes.md  
          fi  
            
          cat release\_notes.md  
        
      \- name: Create Release  
        if: steps.check\_tag.outputs.exists \== 'false'  
        uses: softprops/action-gh-release@v2  
        with:  
          tag\_name: ${{ steps.get\_version.outputs.tag }}  
          name: v${{ steps.get\_version.outputs.version }}  
          body\_path: release\_notes.md  
          draft: false  
          prerelease: false  
          generate\_release\_notes: false  
        env:  
          GITHUB\_TOKEN: ${{ secrets.GITHUB\_TOKEN }}  
        
      \- name: Skip release creation  
        if: steps.check\_tag.outputs.exists \== 'true'  
        run: |  
          echo "Release v${{ steps.get\_version.outputs.version }} already exists. Skipping release creation."


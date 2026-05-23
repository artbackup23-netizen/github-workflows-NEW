name: Build Android APK
on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    
    permissions:
      contents: read
      checks: write
      
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci
        
      - name: Sync Capacitor
        run: npm run mobile:build
        
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'zulu'
          cache: 'gradle'

      - name: Make Gradle executable
        run: chmod +x android/gradlew

      - name: Build Android App (APK)
        working-directory: ./android
        run: ./gradlew assembleDebug --no-daemon
        continue-on-error: true

      - name: Check if APK was built
        id: check_apk
        run: |
          APK_PATH="android/app/build/outputs/apk/debug/app-debug.apk"
          if [ -f "$APK_PATH" ]; then
            APK_SIZE=$(du -h "$APK_PATH" | cut -f1)
            echo "apk_exists=true" >> $GITHUB_OUTPUT
            echo "apk_size=$APK_SIZE" >> $GITHUB_OUTPUT
            echo "✅ APK built successfully - Size: $APK_SIZE"
          else
            echo "apk_exists=false" >> $GITHUB_OUTPUT
            echo "❌ APK file not found at expected location"
            exit 1
          fi

      - name: Upload APK Artifact
        if: steps.check_apk.outputs.apk_exists == 'true'
        uses: actions/upload-artifact@v4
        with:
          name: FinTrack-Pro-APK
          path: android/app/build/outputs/apk/debug/app-debug.apk
          retention-days: 30
          compression-level: 9
          if-no-files-found: error

      - name: Generate Build Summary
        if: always()
        run: |
          echo "## 📱 Build Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "- **Branch**: ${{ github.ref_name }}" >> $GITHUB_STEP_SUMMARY
          echo "- **Commit**: ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY
          echo "- **APK Built**: ${{ steps.check_apk.outputs.apk_exists }}" >> $GITHUB_STEP_SUMMARY
          if [ "${{ steps.check_apk.outputs.apk_exists }}" = "true" ]; then
            echo "- **APK Size**: ${{ steps.check_apk.outputs.apk_size }}" >> $GITHUB_STEP_SUMMARY
          fi
          echo "- **Status**: ${{ job.status }}" >> $GITHUB_STEP_SUMMARY

      - name: Notify on Failure
        if: failure()
        run: |
          echo "::error::Android APK build failed on branch ${{ github.ref_name }}"
          exit 1

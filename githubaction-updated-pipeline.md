name: Deploy Backend to Remote Server

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:

    # Step 1: Checkout Backend code from repository
    - name: Checkout Backend Repository
      uses: actions/checkout@v3

    # Step 2: Print the branch name
    - name: Print Branch
      run: echo "Running on branch ${{ github.ref_name }}"

    # Step 3: Setup Node.js
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '20' # Specify Node.js version 20

    # Step 4: Cache pnpm dependencies
    - name: Cache pnpm modules
      id: cache-pnpm
      uses: actions/cache@v3
      with:
        path: |
          ~/.pnpm-store
          node_modules
        key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
        restore-keys: |
          ${{ runner.os }}-pnpm-
          ${{ runner.os }}-
        
    # Step 5: Install pnpm if cache missed or dependencies changed
    - name: Install pnpm
      run: npm install -g pnpm  

    # Step 6: Install dependencies using pnpm
    - name: Install dependencies
      run: pnpm install

    # Step 7: Build the project (create dist folder)
    - name: Build the project
      run: pnpm build

    # Step 8: Move package.json into the dist folder
    - name: Move package.json into dist folder
      run: mv package.json dist

    # Step 9: Create a zip package of the dist folder named backend.zip
    - name: Create backend.zip package
      run: zip -r backend.zip dist/

    # Step 10: Set up SSH agent with private key
    - name: Set up SSH Key
      uses: webfactory/ssh-agent@v0.5.3
      with:
        ssh-private-key: ${{ secrets.DROPLET_PRIVATE_KEY }}

    # Step 11: Add Remote Server to known_hosts
    - name: Add Remote Server to known_hosts
      run: ssh-keyscan -p ${{ secrets.REMOTE_PORT }} ${{ secrets.REMOTE_IP }} >> ~/.ssh/known_hosts

    # Step 12: Transfer the zip file to the remote server
    - name: Upload Backend Zip to Remote Server
      run: |
        scp -P ${{ secrets.REMOTE_PORT }} ./backend.zip ${{ secrets.REMOTE_USERNAME }}@${{ secrets.REMOTE_IP }}:/home/${{ secrets.REMOTE_USERNAME }}/

    # Step 13: Deploy Backend on Remote Server
    - name: Deploy Backend on Remote Server
      run: |
        ssh -p ${{ secrets.REMOTE_PORT }} ${{ secrets.REMOTE_USERNAME }}@${{ secrets.REMOTE_IP }} << 'EOF'
          #!/bin/bash
          echo "Starting Backend Deployment..."

          BACKEND_PATH="/var/www/backend"
          BACKEND_ZIP="/home/${{ secrets.REMOTE_USERNAME }}/backend.zip"

          # Stop the simply-be service
          pm2 stop simply-be

          # Deploy Backend
          cd $BACKEND_PATH
          sudo rm -rf backend.zip dist/
          sudo mv $BACKEND_ZIP $BACKEND_PATH
          sudo unzip backend.zip
          
          # Setting user permission on dist folder
          sudo chown -R ${{ secrets.REMOTE_USERNAME }} /var/www/backend/dist
          
          # Install Production dependencies
          cd dist
          pnpm i --production
          ls -lrt
          cd ..

          # Restart the service
          pm2 restart simply-be
          
          # Verify deployment status
          pm2 status simply-be
        EOF

// Pipeline voor de .NET todo-app
node {
    stage('Checkout') {
        checkout scm
    }

    stage('Preparation') {
        sh 'docker rm -f todoapp todoappdb || true'
        sh 'docker network create todo-net || true'
    }

    stage('Database') {
        sh '''
            docker run -d --name todoappdb --network todo-net \
              -e MARIADB_ROOT_PASSWORD=sekrit \
              -e MARIADB_DATABASE=todo_db \
              -e MARIADB_USER=todo_usr \
              -e MARIADB_PASSWORD=letmeinplz \
              -v todo-db-data:/var/lib/mysql \
              --health-cmd "healthcheck.sh --connect --innodb_initialized" \
              --health-interval 5s --health-retries 20 \
              mariadb:11

            # Wachten tot de database klaar is
            for i in $(seq 1 30); do
              STATUS=$(docker inspect -f '{{.State.Health.Status}}' todoappdb)
              echo "Database status: ${STATUS}"
              [ "${STATUS}" = "healthy" ] && break
              sleep 2
            done

            # Tabel aanmaken
            docker exec -i todoappdb mariadb -u todo_usr -pletmeinplz todo_db < TodoApp/schema.sql
        '''
    }

    stage('Build') {
        sh 'docker build -t todoapp ./TodoApp'
    }

    stage('Deploy') {
        sh '''
            docker run -d --name todoapp --network todo-net -p 8081:8080 \
              -e "ConnectionStrings__TodoDb=Server=todoappdb;Port=3306;Database=todo_db;User=todo_usr;Password=letmeinplz;" \
              -e ASPNETCORE_ENVIRONMENT=Development \
              todoapp
        '''
    }

    stage('Test') {
        sh '''
            for i in $(seq 1 15); do
              if curl -sf http://172.16.0.10:8081/ > /dev/null; then
                echo "Test geslaagd: app bereikbaar op poort 8081"
                exit 0
              fi
              sleep 2
            done
            echo "Test gefaald, logs van de app:"
            docker logs todoapp
            exit 1
        '''
    }
}

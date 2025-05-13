# GOS_EXAM
https://github.com/NikitaChernikov04/DE/tree/improved-design


## Для иморта в бд



      from flask import Flask, request, jsonify
      import pandas as pd
      from sqlalchemy.orm import sessionmaker
      from models import MyData, Base, engine
      from werkzeug.utils import secure_filename
      import os
      
      app = Flask(__name__)
      
      # Настройки для загрузки файлов
      UPLOAD_FOLDER = 'uploads'
      ALLOWED_EXTENSIONS = {'xlsx'}
      app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
      
      # Создаем таблицы в базе данных
      Base.metadata.create_all(bind=engine)
      
      # Создаем сессию для работы с БД
      Session = sessionmaker(bind=engine)
      
      def allowed_file(filename):
          return '.' in filename and \
                 filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS
      
      @app.route('/upload-excel', methods=['POST'])
      def upload_excel():
          # Проверяем, есть ли файл в запросе
          if 'file' not in request.files:
              return jsonify({"error": "Файл не найден в запросе"}), 400
          
          file = request.files['file']
          
          # Проверяем, что файл имеет допустимое расширение
          if file.filename == '':
              return jsonify({"error": "Не выбран файл"}), 400
          
          if not allowed_file(file.filename):
              return jsonify({"error": "Только файлы .xlsx поддерживаются"}), 400
          
          # Сохраняем файл во временную папку
          if not os.path.exists(app.config['UPLOAD_FOLDER']):
              os.makedirs(app.config['UPLOAD_FOLDER'])
          
          filename = secure_filename(file.filename)
          filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
          file.save(filepath)
          
          try:
              # Читаем Excel-файл во временный DataFrame
              df = pd.read_excel(filepath, engine='openpyxl')
              
              # Создаем сессию БД
              db = Session()
              
              # Преобразуем строки в объекты MyData
              for _, row in df.iterrows():
                  entry = MyData(name=row['name'], value=row['value'])
                  db.add(entry)
              
              db.commit()
              
              return jsonify({
                  "status": "Импорт завершён",
                  "rows_imported": len(df)
              })
          
          except Exception as e:
              return jsonify({"error": str(e)}), 500
          
          finally:
              # Удаляем временный файл
              if os.path.exists(filepath):
                  os.remove(filepath)
              
              # Закрываем сессию БД
              db.close()
      
      if __name__ == '__main__':
          app.run(debug=True)


## er-diagram

    SELECT table_name FROM information_schema.tables WHERE table_schema='public';

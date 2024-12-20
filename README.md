import numpy as np
import tensorflow as tf
from tensorflow.keras.applications import MobileNetV2
from tensorflow.keras.models import Model
from tensorflow.keras.layers import Dense, Flatten, Dropout
from pathlib import Path
from PIL import Image
import csv
import os
import matplotlib.pyplot as plt
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4

#LBL1: Загрузка данных из папки
def load_dataset_from_directory(directory: Path):
    frames, labels, coords = [], [], []
    for root, _, files in os.walk(directory):
        for file in files:
            if file.endswith('.jpg'):
                img = Image.open(os.path.join(root, file)).resize((224, 224))
                frames.append(np.array(img, dtype=np.uint8))
            elif file == 'labels.csv':
                with open(os.path.join(root, file), newline='') as csvfile:
                    reader = csv.reader(csvfile)
                    next(reader)
                    for row in reader:
                        try:
                            labels.append(int(row[1]) if row[1] else 0)
                            coords.append([int(row[2]) if row[2] else 0, int(row[3]) if row[3] else 0])
                        except ValueError:
                            print(f"Invalid value in row: {row}")
    return np.array(frames), np.array(labels), np.array(coords)

#LBL2: Создание модели
def initialize_model():
    base_model = MobileNetV2(weights='imagenet', include_top=False, input_shape=(224, 224, 3))
    base_model.trainable = False
    x = Dropout(0.5)(Flatten()(base_model.output))
    model = Model(inputs=base_model.input, outputs=[Dense(1, activation='sigmoid', name='classification')(x),
                                                    Dense(2, activation='linear', name='regression')(x)])
    return model

# Компиляция модели
model = initialize_model()
model.compile(optimizer=tf.keras.optimizers.Adam(learning_rate=0.001),
              loss={'classification': 'binary_crossentropy', 'regression': 'mse'},
              loss_weights={'classification': 1.0, 'regression': 0.5}, metrics={'classification': 'accuracy'})

#LBL1: Тренировка модели
def fit_model(model, train_data, val_data, epochs=10, batch_size=32):
    train_frames, train_labels, train_coords = train_data
    val_frames, val_labels, val_coords = val_data
    train_dataset = tf.data.Dataset.from_tensor_slices(
        (train_frames, {'classification': train_labels, 'regression': train_coords})).map(
        lambda x, y: (tf.image.resize(tf.cast(x, tf.float32) / 255.0, (224, 224)), y)).batch(batch_size)
    val_dataset = tf.data.Dataset.from_tensor_slices(
        (val_frames, {'classification': val_labels, 'regression': val_coords})).map(
        lambda x, y: (tf.image.resize(tf.cast(x, tf.float32) / 255.0, (224, 224)), y)).batch(batch_size)

    history = model.fit(train_dataset, validation_data=val_dataset, epochs=epochs)

    # Создание графиков
    plt.figure(figsize=(12, 5))

    # График потерь
    plt.subplot(1, 2, 1)
    plt.plot(history.history['loss'], label='Train Loss')
    plt.plot(history.history['val_loss'], label='Val Loss')
    plt.xlabel('Epochs')
    plt.ylabel('Loss')
    plt.legend()
    plt.title('Loss Over Epochs')

    # График точности
    plt.subplot(1, 2, 2)
    plt.plot(history.history['classification_accuracy'], label='Train Accuracy')
    plt.plot(history.history['val_classification_accuracy'], label='Val Accuracy')
    plt.xlabel('Epochs')
    plt.ylabel('Accuracy')
    plt.legend()
    plt.title('Accuracy Over Epochs')

    plt.savefig('training_results.png')
    plt.show()

    return history

#LBL2: Расчет Simple Ball Tracking Accuracy (SiBaTrAcc)
def compute_sibatracc(predictions, ground_truth, e1=5, e2=5, step=8, alpha=1.5):
    def calculate_error(code_pr, x_pr, y_pr, code_gt, x_gt, y_gt):
        if code_gt != 0 and code_pr == 0:
            return e1
        elif code_gt == 0 and code_pr != 0:
            return e2
        else:
            distance = np.sqrt((x_gt - x_pr) ** 2 + (y_gt - y_pr) ** 2)
            return min(5, distance / step) ** alpha

    total_error = 0
    N = len(predictions)
    for pred, gt in
# KRHF
Kognitív robotika HF

Feladatunk vobalkövetés megvalósítása volt turtlebottal és neurűlis hálóval.
A neurális hálót a roboton kellett futtatnunk egy külön számítógép helyett.
Ennek érdekében minél kisebb és gyorsabb, de a feladat megvalósításához még megfelelő méretű neurális háló létrehozására törekedtünk.
Az elkészült vonalkövető rendszert a valós roboton kellett tesztelnünk és bemutatnunk.

A feladat megvalósítását a kis méretű neurális háló létrehozásával kezdtük. 
Az elkészült neurális hálót kezdetben a szimulációs környezetben teszteltük a gyakorlatokon tanultaknak megfelelően.
A robot a szimulációs környezetben megfelelően működött, követte a vonalat.
Ezután áttértünk a valós roboton való tesztelésre. 
Ekkor több problémába is ütköztünk. A robottal való kommunikációs problémák megoldása után kiderült, hogy a Tensorflow könyvtárat nem tudjuk telepíteni a robotra,
méretproblémák miatt. Így a Tensorflow Lite verziót kellett alkalmaznunk, amihez a már elkészített modellt is át kellett alakítanunk ennek megfelelően.
Ezzel párhuzamosan leteszteltük a neurális háló működését, ami a szimulációs környezethez képest sokkal rosszabb eredményeket mutatott.
A valós robottal több, mint 100 képet készítettünk a valós környezetről, annak érdekében, hogy a neurális hálónk jobban tudjon alkalmazkodni a valós helyzethez.
Így egy újabb tanítás és neurális háló egyszerűsítés következett.
Az új, elkészült modellt átalakítottuk a Tensorflow Lite könyvtárnak megfelelően, és ismét szimulációban teszteltük a működését.
Ezután a valós roboton is leteszteltük az elkészült vonalkövető rendszert.



### Telepítési útmutató
Első lépésként hozzuk létre a csomagunkat tartalmazó workspace-t. Majd a catkin_make paranccsal fordítsuk.
```
cd ~
mkdir -p catkin_ws/src
cd catkin_ws
catkin_make
```
Ezt követően sorce-olható a workspace.
```
source ~/catkin_ws/devel/setup.bash
```
Ezt betehetjük a .bashrc fájlba is.
```
cd ~
nano .bashrc
--------------------------------------------------
WORKSPACE=~/catkin_ws/devel/setup.bash
source $WORKSPACE
```
Ezt követően töltsük le a MOGI turtlebot3 verziót, majd a ```mogi-ros``` branch-re lépjünk át.
```
git clone https://github.com/MOGI-ROS/turtlebot3
git checkout mogi-ros
````
?? itt kell catkin_ws??

Ezt követően töltsük le a line_follower_cnn.py és model.best.h5 fájlokat és cseréljük le a mogi turtlebot3 meglévő fájljait.

A robotra való bejeltkezést ssh-val oldjuk meg. Ehhez szükséges, hogy azonos hálón legyünk a robottal. A laborban lévő 2 robot ip címe: ```192.168.50.111``` valamint ```192.168.50.175```, ezek közül az egyikre lesz szükségünk.
```
ssh 192.168.50.111 -lubuntu
vagy
ssh 192.168.50.175 -lubuntu
pw: ubuntu
```
A turtlebot véges memória kapacitása miatt tensorflow lite-ot kell telepíteni, és ezzel futtatható a tanított neurális háló. 
```
python3 -m pip install tflite-runtime
```


# ettől
A neurális háló futtatásához, először tanítani kell ezt. Ehhez képeket készítettünk, majd ezeket osztályoztuk és tanítottuk a CNN-t.
A képek készítéshez először indítsuk el a roboton a bringup fájlt.
```
roslaunch turtlebot3_bringup turtlebot3_robot.launch
```
Ezt követően indítsunk egy másik terminált, amelyben állítsuk be ROS Mastert, ezt az ssh után kapott üzenetből tudjuk kimásolni.
```
export ROS_MASTER_URI=http://192.168.1.111:11311
```
A képek készítését a ```save_training_images.py``` végzi el.

```
rosrun turtlebot3_mogi save_training_images.py
```
Amennyiben egyedül vagyunk, akkor egy újabb terminálban a teleop indításával megoldható, hogy a robotot irányítsuk, átmozgasssuk.
```
roslaunch turtlebot3_teleop turtlebot3_teleop_key.launch
```

Kellő mennyiségű kép készítését követően osztályozni kell, majd pedig tanítani a modellt. Ezt a számítógépen végezzük, mert a robot tárhely/RAM kapacitása kisebb egy személyi számítógéphez képest kevés. Emiatt a ```line_follower_cnn.py``` fájlt is módosítani kell tensorflow lite csomagra, a modellt is át kell alakítani.

### A modell átkonvertálása
Mivel TensorFlow helyett csak a TensorFlow Lite verziót tudjuk használni, ezért szükséges a modellt átkonvertálnunk. Ehhez létrehoztunk egy külön lefuttatható Python scriptet, amely segítségével a korábban generált Keras modellből TFLite modellt készítünk a megfelelő path-ok megadásával:

```python
import tensorflow as tf

# Load the original Keras model
model = tf.keras.models.load_model('/home/catkin_ws/src/turtlebot3/turtlebot3_mogi/network_model/model.best.h5')

# Convert the model to TensorFlow Lite
converter = tf.lite.TFLiteConverter.from_keras_model(model)
tflite_model = converter.convert()

# Save the TFLite model.
with open('/home/catkin_ws/src/turtlebot3/turtlebot3_mogi/network_model/model.tflite', 'wb') as f:
    f.write(tflite_model)
```

### Új line follower node létrehozása
A vonalkövetés megvalósítását a TensorFlow Lite csomag segítségével végezzük. Emiatt az órán megírt ```line_follower_cnn.py``` fájt több helyen is módosítanunk kellett.

A több, TensorFlow-hoz szükséges csomag helyett elegendő a TFLite-ot importálni:
```python
import tflite_runtime.interpreter as tflite
```

TensorFlow esetén elég volt betölteni a modellt a ```load_model``` segítségével, a predikció pedig a ```model(image)``` segítségével történt. Azonban TensorFLow Lite használata esetén létre kellett hozni egy ```Interpreter``` objektumot, allokálni a tenzorokat, majd a predikcióhoz használni a ```set_tensor``` és ```invoke``` függvényeket.

```python
# Load TFLite model and allocate tensors
interpreter = tflite.Interpreter(model_path=model_path)
interpreter.allocate_tensors()

# Get input and output tensors.
input_details = interpreter.get_input_details()
output_details = interpreter.get_output_details()
```

A képfeldolgozó függvény módosítása is szükséges volt. A képet array-é alakítás után 4D-ssé formáztuk, mivel a TensorFlow Lite modellnek szüksége van a batch dimenzió megadására is a futáshoz.
```python
def processImage(self, img):
        image = cv2.resize(img, (image_size, image_size))
        image = np.reshape(image, (-1, image_size, image_size, 3)).astype("float32") / 255.0

        # Set the value of the input tensor
        interpreter.set_tensor(input_details[0]['index'], image)
        interpreter.invoke()

        # Retrieve the value of the output tensor
        prediction = np.argmax(interpreter.get_tensor(output_details[0]['index']))
        
        [...]
```

Ezek a legfőbb módosítások, a többi (pl. verziók kiíratása és ellenőrzése) megtalálható a feltöltött ```line_follower_cnn.py``` fájlban.


# eddig újragondolni









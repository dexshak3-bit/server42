# server42


import os
import json
import pathlib
import random

from fastapi import FastAPI, Header, HTTPException
from fastapi.params import Depends
from pydantic import BaseModel, Field
import uvicorn
from datetime import date

app = FastAPI(title="autopark")

filePath = pathlib.Path("data")
driversFile = filePath/"drivers.json"
carsFile = filePath/"cars.json"
parkingFile = filePath/"parking.json"


adminLogin = "admin1"
adminPassword = "1234"
adminToken = "qwerty"

class AdminLoginRequest(BaseModel):
    login: str
    password: str

class AdminLoginResponse(BaseModel):
    token: str

class CarCreateRequest(BaseModel):
    vehicle_id: int
    brand: str
    model: str
    license_plate: str
    year: str
    color: str

class DriverCreateRequest(BaseModel):
    driverid: int
    fio: str
    phone: str
    category: str

class ParkingCreateRequest(BaseModel):
    driverid: int
    carid: int
    parkingid: int
    start_date: date
    cost: str

def loadfile(path: pathlib.Path):
    if not path.exists():
        return []
    text = path.read_text(encoding="Utf-8").strip()
    return json.loads(text) if text else []

def savefile(path: pathlib.Path, data):
    path.write_text(json.dumps(data, ensure_ascii=False), encoding="Utf-8")

def checkadmin(x_admin_token: str | None = Header(None, alias="X-Admin-Token")):
    if x_admin_token !=adminToken:
        raise HTTPException(status_code=401, detail="Токен не соответсвует")

@app.get("/cars")
def getcars():
    return loadfile(carsFile)

@app.get("/drivers")
def getdrivers():
    return loadfile(driversFile)

@app.get("/parking")
def getparking():
    return loadfile(parkingFile)

@app.post("/login/admin", response_model=AdminLoginResponse)
def authorizateadmin(request: AdminLoginRequest):
    if request.login != adminLogin or request.password != adminPassword:
        raise HTTPException(status_code=401, detail="неверный логин или пароль")
    return AdminLoginResponse(token= adminToken)

@app.post("/cars", response_model=CarCreateRequest, dependencies=[Depends(checkadmin)])
def addnewcard(request: CarCreateRequest):
    cars = loadfile(carsFile)
    cars.append(request.model_dump())
    savefile(carsFile, cars)
    return request

@app.post("/drivers", response_model=DriverCreateRequest, dependencies=[Depends(checkadmin)])
def addnewdriver(request: DriverCreateRequest):
    drivers = loadfile(driversFile)
    existing_id = {i.get("driverid") for i in drivers if i.get("driverid") is not None}
    while True:
        new_id = random.randint(10000, 99999)
        if new_id not in existing_id:
            break

    new_driver = request.model_dump()
    new_driver["driverid"] = new_id
    drivers.append(new_driver)
    savefile(driversFile, drivers)
    return new_driver

@app.post("/parking", response_model=ParkingCreateRequest, dependencies=[Depends(checkadmin)])
def addnewparking(request: ParkingCreateRequest):
    parkings = loadfile(parkingFile)

    parking = {
        "driverid": request.driverid,
        "carid": request.carid,
        "parkingid": request.parkingid,
        "start_date": request.start_date.isoformat(),
        "cost": request.cost
    }

    parkings.append(parking)
    savefile(parkingFile, parkings)
    return parking

if __name__ == "__main__":
    uvicorn.run(app, host="127.0.0.1", port=8000)







    ApiClient:



using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Cache;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;
using System.Text.Json;

namespace exam9
{
    public class ApiClient
    {
        private readonly HttpClient _htppClient;
        private string _adminToken;

        private static readonly JsonSerializerOptions JsonOpts = new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true
        };
        public ApiClient(string baseUrl = "http://127.0.0.1:8000")
        {
            _htppClient = new HttpClient()
            {
                BaseAddress = new Uri(baseUrl)
            };
        }
        public void SetAdminToken(string token)
        {
            _adminToken = token;
            _htppClient.DefaultRequestHeaders.Clear();
            if (!string.IsNullOrEmpty(_adminToken))
            {
                _htppClient.DefaultRequestHeaders.Add("X-Admin-Token", _adminToken);
            }    
        }
        public async Task<bool> AdminLoginAsync(string login, string password)
        {
            var req = new AdminLoginRequest { login = login, password = password };
            var json = JsonSerializer.Serialize(req, JsonOpts);
            var content = new StringContent(json, Encoding.UTF8, "application/json");

            var response = await _htppClient.PostAsync("/login/admin", content);
            if (!response.IsSuccessStatusCode)
            {
                throw new Exception("Ошибка авторизации");
            }
            var responseJson = await response.Content.ReadAsStringAsync();
            var result = JsonSerializer.Deserialize<AdminLoginResponse>(responseJson, JsonOpts);
            if (result == null || string.IsNullOrEmpty(result.token))
            {
                throw new Exception("Сервер вернул пустой токен");
            }
            SetAdminToken(result.token);
            return true;
        }
        public async Task<CarCreateRequest> AddCarAsync(CarCreateRequest car)
        {
            var json = JsonSerializer.Serialize(car, JsonOpts);
            var content = new StringContent(json, Encoding.UTF8, "application/json");

            var response = await _htppClient.PostAsync("/cars", content);
            if (!response.IsSuccessStatusCode)
            {
                throw new Exception("Ошибка добавления машины");
            }
            var responseJson = await response.Content.ReadAsStringAsync();
            return  JsonSerializer.Deserialize<CarCreateRequest>(responseJson, JsonOpts) ?? car;
        }
        public async Task<List<CarCreateRequest>> GetCarAsync()
        {

            var response = await _htppClient.GetAsync("/cars");
            if (!response.IsSuccessStatusCode)
            {
                throw new Exception("Ошибка загрузки машины");
            }
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<List<CarCreateRequest>>(responseJson, JsonOpts) ?? new List<CarCreateRequest>();
        }
                
        public async Task<DriverCreateRequest> AddDriverAsync(DriverCreateRequest driver)
        {
            var json = JsonSerializer.Serialize(driver, JsonOpts);
            var content = new StringContent(json, Encoding.UTF8, "application/json");

            var response = await _htppClient.PostAsync("/drivers", content);
            if (!response.IsSuccessStatusCode)
            {
                throw new Exception("Ошибка добавления водителя");
            }
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<DriverCreateRequest>(responseJson, JsonOpts) ?? driver;
        }
        public async Task<List<DriverCreateRequest>> GetDriverAsync()
        {

            var response = await _htppClient.GetAsync("/drivers");
            if (!response.IsSuccessStatusCode)
            {
                throw new Exception("Ошибка загрузки водителя");
            }
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<List<DriverCreateRequest>>(responseJson, JsonOpts) ?? new List<DriverCreateRequest>();
        }
        public async Task<ParkingCreateRequest> AddParkingAsync(ParkingCreateRequest parking)
        {
            var json = JsonSerializer.Serialize(parking, JsonOpts);
            var content = new StringContent(json, Encoding.UTF8, "application/json");

            var response = await _htppClient.PostAsync("/parking", content);
            if (!response.IsSuccessStatusCode)
            {
                throw new Exception("Ошибка добавления парковачных мест");
            }
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<ParkingCreateRequest>(responseJson, JsonOpts) ?? parking;
        }
        public async Task<List<ParkingCreateRequest>> GetParkingAsync()
        {

            var response = await _htppClient.GetAsync("/parking");
            if (!response.IsSuccessStatusCode)
            {
                throw new Exception("Ошибка загрузки парковачных мест");
            }
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<List<ParkingCreateRequest>>(responseJson, JsonOpts) ?? new List<ParkingCreateRequest>();
        }
    }
}



    Models:



using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace exam9
{
    public static class ApplicationState
    {
        public static ApiClient ApiClient { get; set; }
    }
    public class AdminLoginRequest
    {
        public string login { get; set; }
        public string password { get; set; }
    }
    public class AdminLoginResponse
    {
        public string token { get; set; }
    }
    public class CarCreateRequest
    {
        public int vehicle_id { get; set; }
        public string brand { get; set; }
        public string model { get; set; }
        public string license_plate { get; set; }
        public string year { get; set; }
        public string color { get; set; }
    }
    public class DriverCreateRequest
    {
        public int driverid { get; set; }
        public string fio { get; set; }
        public string phone { get; set; }
        public string category { get; set; }
    }
    public class ParkingCreateRequest
    {
        public int driverid { get; set; }
        public int carid { get; set; }
        public int parkingid { get; set; }
        public string start_date { get; set; }
        public string cost { get; set; }
    }

}



     MainWindow Grid:


    <Grid>
    <TextBlock Text="Окно авторизации" FontSize="22" TextAlignment="Center" Margin="282,0,282,333" Height="39" VerticalAlignment="Bottom"></TextBlock>
    <TextBox x:Name="logintxt" Margin="282,0,251,252" BorderBrush="Black" Height="51" VerticalAlignment="Bottom"></TextBox>
    <TextBox x:Name="passwordtxt" Margin="282,0,251,166" BorderBrush="Black" Height="51" VerticalAlignment="Bottom"></TextBox>
    <Button x:Name="authorizatebutton" Content="Войти" BorderBrush="Black" Background="Black" Foreground="White" Margin="350,0,319,83" Click="authorizatebutton_Click" Height="45" VerticalAlignment="Bottom"></Button>
    <TextBlock Text="Логин:" FontSize="14" TextAlignment="Center" Margin="208,0,523,258" Height="39" VerticalAlignment="Bottom"></TextBlock>
    <TextBlock Text="Пароль:" FontSize="14" TextAlignment="Center" Margin="213,0,528,167" Height="40" VerticalAlignment="Bottom"></TextBlock>
    </Grid>



      c# MainWindow:

      
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Navigation;
using System.Windows.Shapes;

namespace exam9
{
    /// <summary>
    /// Логика взаимодействия для MainWindow.xaml
    /// </summary>
    public partial class MainWindow : Window
    {
        private readonly ApiClient _apiClient;
        public MainWindow()
        {
            InitializeComponent();
            _apiClient = new ApiClient();

        }

        private async void authorizatebutton_Click(object sender, RoutedEventArgs e)
        {
            try
            {
                await _apiClient.AdminLoginAsync(logintxt.Text, passwordtxt.Text);
                ApplicationState.ApiClient = _apiClient;
                MessageBox.Show("Добро пожаловать!");
                Window1 a = new Window1();
                a.Show();
                this.Close();
            }
            catch(Exception ex)
            {
                MessageBox.Show(ex.Message);
            }
        }
    }
}


        Window1 Grid:
        <Grid>
        <Button x:Name="addnewdriverbutton" Content="Добавить нового пользователя" Margin="156,0,127,291" Height="60" VerticalAlignment="Bottom" Click="addnewdriverbutton_Click"></Button>
        <Button x:Name="addnewcarbutton" Content="Добавить новый автомобилб" Margin="156,0,127,192" Height="60" VerticalAlignment="Bottom" Click="addnewcarbutton_Click"></Button>
        <Button x:Name="addnewparkingbutton" Content="Выдать новое парковочное место" Margin="156,0,127,83" Height="60" VerticalAlignment="Bottom" Click="addnewparkingbutton_Click"></Button>
        </Grid>







    Window1 C#:
    using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Shapes;

namespace exam9
{
    /// <summary>
    /// Логика взаимодействия для Window1.xaml
    /// </summary>
    public partial class Window1 : Window
    {
        public Window1()
        {
            InitializeComponent();
        }

        private void addnewdriverbutton_Click(object sender, RoutedEventArgs e)
        {
            Window2 a = new Window2();
            a.Show();
            this.Close();
        }

        private void addnewcarbutton_Click(object sender, RoutedEventArgs e)
        {
            Window3 a = new Window3();
            a.Show();
            this.Close();
        }

        private void addnewparkingbutton_Click(object sender, RoutedEventArgs e)
        {
            Window4 a = new Window4();
            a.Show();
            this.Close();
        }
    }
}






        Window2 Grid:
        <Grid>
    <Button x:Name="exitbutton" Content="Выйти" Height="45" VerticalAlignment="Top" HorizontalAlignment="Left" Width="115" Click="exitbutton_Click"/>
    <DataGrid x:Name="driverslist" Margin="177,0,44,242" Height="170" VerticalAlignment="Bottom"></DataGrid>
    <TextBox x:Name="fiotxt" Margin="266,0,238,183" BorderBrush="Black" Height="34" VerticalAlignment="Bottom">
    </TextBox>
    <TextBox x:Name="phonetxt" MaxLength="11" Margin="266,0,238,128" BorderBrush="Black" Height="34" VerticalAlignment="Bottom"></TextBox>
    <TextBox x:Name="categorytxt" Margin="266,0,238,71" BorderBrush="Black" Height="34" VerticalAlignment="Bottom"></TextBox>
    <Button x:Name="addnewdriverbutton" Content="Добавить" BorderBrush="Black" Foreground="White" Background="Black" Margin="301,0,273,10" Height="45" VerticalAlignment="Bottom" Click="addnewdriverbutton_Click" ></Button>
    <TextBlock Text="Список водителей:" Margin="408,0,276,412" Height="17" VerticalAlignment="Bottom"></TextBlock>
    <TextBlock Text="ФИО:" Margin="266,0,418,220" Height="17" VerticalAlignment="Bottom"></TextBlock>
    <TextBlock Text="Телефон:" Margin="266,0,418,161" Height="17" VerticalAlignment="Bottom"></TextBlock>
    <TextBlock Text="Категория:" Margin="266,0,418,106" Height="17" VerticalAlignment="Bottom"></TextBlock>
    </Grid>






    Window2 C#:


    using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Shapes;

namespace exam9
{
    /// <summary>
    /// Логика взаимодействия для Window2.xaml
    /// </summary>
    public partial class Window2 : Window
    {
        public Window2()
        {
            InitializeComponent();
            LoadDrivers();
        }

        private async void addnewdriverbutton_Click(object sender, RoutedEventArgs e)
        {
            if (string.IsNullOrEmpty(fiotxt.Text) || string.IsNullOrEmpty(phonetxt.Text) || string.IsNullOrEmpty(categorytxt.Text))
            {
                MessageBox.Show("Заполните все поля");
            }
            var req = new DriverCreateRequest
            {
                fio = fiotxt.Text.Trim(),
                phone = phonetxt.Text.Trim(),
                category = categorytxt.Text.Trim()
            };
            try
            {
                var created = await ApplicationState.ApiClient.AddDriverAsync(req);
                MessageBox.Show("водитель добавлен");
                fiotxt.Text = "";
                phonetxt.Text = "";
                categorytxt.Text = "";
                LoadDrivers();
            }
            catch(Exception ex)
            {
                MessageBox.Show(ex.Message);
            }
        }
        private async void LoadDrivers()
        {
            try
            {
                var driver = await ApplicationState.ApiClient.GetDriverAsync();
                driverslist.ItemsSource = driver;
            }
            catch(Exception ex)
            {
                MessageBox.Show(ex.Message);
            }
        }

        private void exitbutton_Click(object sender, RoutedEventArgs e)
        {
            Window1 a = new Window1();
            a.Show();
            this.Close();
        }
    }
}







        Window3 Grid:
         <Grid>
     <Button x:Name="exitbutton" Content="Выйти" Height="40" VerticalAlignment="Top" HorizontalAlignment="Left" Width="91" Click="exitbutton_Click"/>
     <DataGrid x:Name="carslist" Margin="165,0,71,274" Height="140" VerticalAlignment="Bottom"></DataGrid>
     <TextBox x:Name="brandtxt" Margin="76,0,448,202" BorderBrush="Black" Height="35" VerticalAlignment="Bottom"></TextBox>
     <TextBox x:Name="modeltxt" Margin="76,0,448,143" BorderBrush="Black" Height="35" VerticalAlignment="Bottom"></TextBox>
     <TextBox x:Name="licenseplatetxt" Margin="414,0,110,202" BorderBrush="Black" Height="35" VerticalAlignment="Bottom"></TextBox>
     <TextBox x:Name="colortxt" Margin="262,0,262,72" BorderBrush="Black" Height="35" VerticalAlignment="Bottom"></TextBox>
     <TextBox x:Name="yeartxt" MaxLength="4" Margin="414,0,110,143" BorderBrush="Black" Height="35" VerticalAlignment="Bottom"></TextBox>
     <Button x:Name="addnewcarbutton" Content="Добавить" Margin="302,0,320,10" BorderBrush="Black" Background="Black" Foreground="White" Height="40" VerticalAlignment="Bottom" Click="addnewcarbutton_Click" ></Button>
     <TextBlock Text="Список машин:" Margin="396,0,302,413" Height="21" VerticalAlignment="Bottom"></TextBlock>
     <TextBlock Text="Марка машины:" Margin="76,0,600,237" Height="21" VerticalAlignment="Bottom"></TextBlock>
     <TextBlock Text="Цвет машины:" Margin="262,0,414,107" Height="21" VerticalAlignment="Bottom"></TextBlock>
     <TextBlock Text="Модель машины:" Margin="76,0,600,178" Height="21" VerticalAlignment="Bottom"></TextBlock>
     <TextBlock Text="Гос-номер машины:" Margin="414,0,262,239" Height="21" VerticalAlignment="Bottom"></TextBlock>
     <TextBlock Text="Год-выпуска машины:" Margin="414,0,262,176" Height="21" VerticalAlignment="Bottom"></TextBlock>
     </Grid>




        Window3 C#:


        using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Shapes;

namespace exam9
{
    /// <summary>
    /// Логика взаимодействия для Window3.xaml
    /// </summary>
    public partial class Window3 : Window
    {
        public Window3()
        {
            InitializeComponent();
            LoadCars();
        }

        private async void addnewcarbutton_Click(object sender, RoutedEventArgs e)
        {
            if (string.IsNullOrEmpty(brandtxt.Text) || string.IsNullOrEmpty(modeltxt.Text) || string.IsNullOrEmpty(colortxt.Text) || string.IsNullOrEmpty(yeartxt.Text)|| string.IsNullOrEmpty(licenseplatetxt.Text))
            {
                MessageBox.Show("Заполните все поля");
            }
            try
            {
                var cars = await ApplicationState.ApiClient.GetCarAsync();
                int maxId = 0;
                if (cars !=null && cars.Count > 0)
                {
                    maxId = cars.Max(u => u.vehicle_id);
                }
                int newid = maxId + 1;
                var req = new CarCreateRequest
                {
                    vehicle_id = newid,
                    brand = brandtxt.Text.Trim(),
                    model = modeltxt.Text.Trim(),
                    license_plate = licenseplatetxt.Text.Trim(),
                    year = yeartxt.Text.Trim(),
                    color = colortxt.Text.Trim()
                };
                var created = await ApplicationState.ApiClient.AddCarAsync(req);
                MessageBox.Show("Машина добавлена");
                brandtxt.Text = "";
                modeltxt.Text = "";
                licenseplatetxt.Text = "";
                yeartxt.Text = "";
                colortxt.Text = "";
                LoadCars();
            }
            catch(Exception ex)
            {
                MessageBox.Show(ex.Message);
            }
        }
        private async Task LoadCars()
        {
            try
            {
                var cars = await ApplicationState.ApiClient.GetCarAsync();
                carslist.ItemsSource = cars;
            }
            catch(Exception ex)
            {
                MessageBox.Show(ex.Message);
            }
        }

        private void exitbutton_Click(object sender, RoutedEventArgs e)
        {
            Window1 a = new Window1();
            a.Show();
            this.Close();
        }
    }
}







        Window4 Grid:
        <Window x:Class="exam9.Window4"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:exam9"
        mc:Ignorable="d"
        Title="Окно оформления парковочного места" Height="550" Width="850">
    <Grid>
        <Button x:Name="exitbutton" Content="Выйти" Height="47" VerticalAlignment="Top" HorizontalAlignment="Left" Width="105" Click="exitbutton_Click"/>
        <DataGrid x:Name="parkinglist" Margin="155,0,49,370" Height="140" VerticalAlignment="Bottom"></DataGrid>
        <DataGrid x:Name="driverslist" Margin="33,0,392,199" Height="111" VerticalAlignment="Bottom"></DataGrid>
        <DataGrid x:Name="carslist" Margin="32,0,392,44" Height="119" VerticalAlignment="Bottom"></DataGrid>
        <TextBox x:Name="driveridtxt" Margin="502,0,84,314" BorderBrush="Black" Height="35" VerticalAlignment="Bottom"></TextBox>
        <TextBox x:Name="carirdtxt" Margin="502,0,84,254" BorderBrush="Black" Height="36" VerticalAlignment="Bottom"></TextBox>
        <TextBox x:Name="costtxt" Margin="502,0,84,136" BorderBrush="Black" Height="36" VerticalAlignment="Bottom"></TextBox>
        <DatePicker x:Name="startdatetxt" Margin="502,0,168,191" Height="38" VerticalAlignment="Bottom"></DatePicker>
        <Button x:Name="addnewparkingbutton" Content="Добавить" Foreground="White" BorderBrush="Black" Background="Black" Margin="512,0,94,75" Height="48" VerticalAlignment="Bottom" Click="addnewparkingbutton_Click"></Button>
        <TextBlock Text="Список арендованных парковчаных мест" Margin="356,0,250,505" Height="24" VerticalAlignment="Bottom"></TextBlock>
        <TextBlock Text="Список машин:" Margin="33,0,706,163" Height="27" VerticalAlignment="Bottom">
        </TextBlock>
        <TextBlock Text="Список пользователей:" Margin="33,0,678,304" Height="27" VerticalAlignment="Bottom">
        </TextBlock>
        <TextBlock Text="id Водителя:" Margin="502,0,261,349" Height="16" VerticalAlignment="Bottom"></TextBlock>
        <TextBlock Text="id Машины:" Margin="502,0,261,294" Height="16" VerticalAlignment="Bottom"></TextBlock>
        <TextBlock Text="Дата начала аренды:" Margin="502,0,218,228" Height="17" VerticalAlignment="Bottom"></TextBlock>
        <TextBlock Text="Цена Аренды:" Margin="502,0,261,176" Height="17" VerticalAlignment="Bottom"></TextBlock>

    </Grid>
</Window>




            Window4 C#:


using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Shapes;

namespace exam9
{
    /// <summary>
    /// Логика взаимодействия для Window4.xaml
    /// </summary>
    public partial class Window4 : Window
    {
        public Window4()
        {
            InitializeComponent();
            LoadDrivers();
            LoadCars();
            LoadParking();
            startdatetxt.SelectedDate = DateTime.Now;
        }

        private async void addnewparkingbutton_Click(object sender, RoutedEventArgs e)
        {
            if (string.IsNullOrEmpty(driveridtxt.Text)|| string.IsNullOrEmpty(carirdtxt.Text) || string.IsNullOrEmpty(costtxt.Text))
            {
                MessageBox.Show("Заполните все поля");
                return;
            }
            if (!int.TryParse(driveridtxt.Text, out var driverid))
            {
                MessageBox.Show("Id Водителя должно быть числом");
                return;
            }
            if (!int.TryParse(carirdtxt.Text, out var carid))
            {
                MessageBox.Show("Id машины должно быть числом");
                return;
            }
            try
            {
                var parking = await ApplicationState.ApiClient.GetParkingAsync();
                int maxId = 0;
                if (parking !=null && parking.Count > 0)
                {
                    maxId = parking.Max(u => u.parkingid);
                }
                int newId = maxId + 1;
                var drivers = await ApplicationState.ApiClient.GetDriverAsync();
                if (!drivers.Any(u => u.driverid == driverid))
                {
                    MessageBox.Show("Такого водителя нету");
                    return;
                }
                var cars = await ApplicationState.ApiClient.GetCarAsync();
                if (!cars.Any(u => u.vehicle_id == carid))
                {
                    MessageBox.Show("Такой машины нету");
                    return;
                }
                var req = new ParkingCreateRequest
                {
                    parkingid = newId,
                    carid = carid,
                    driverid = driverid,
                    start_date = startdatetxt.SelectedDate.Value.ToString("yyyy-MM-dd"),
                    cost = costtxt.Text.Trim()
                };
                await ApplicationState.ApiClient.AddParkingAsync(req);
                MessageBox.Show("парковочное место арендовано");
                carirdtxt.Text = "";
                driveridtxt.Text = "";
                costtxt.Text = "";
                LoadParking();
            }
            catch(Exception ex)
            {
                MessageBox.Show(ex.Message);
                return;
            }
        }
        private async void LoadDrivers()
        {
            try
            {
                var created = await ApplicationState.ApiClient.GetDriverAsync();
                driverslist.ItemsSource = created;
            }
            catch(Exception ex)
            {
                MessageBox.Show(ex.Message);
                return;
            }
        }
        private async void LoadCars()
        {
            try
            {
                var created = await ApplicationState.ApiClient.GetCarAsync();
                carslist.ItemsSource = created;
            }
            catch(Exception ex)
            {
                MessageBox.Show(ex.Message);
                return;
            }
        }
        private async void LoadParking()
        {
            try
            {
                var created = await ApplicationState.ApiClient.GetParkingAsync();
                parkinglist.ItemsSource = created;
            }
            catch(Exception ex)
            {
                MessageBox.Show(ex.Message);
                return;
            }
        }

        private void exitbutton_Click(object sender, RoutedEventArgs e)
        {
            Window1 a = new Window1();
            a.Show();
            this.Close();
        }
    }
}

       

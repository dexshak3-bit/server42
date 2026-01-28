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

if name == "main":
    uvicorn.run(app, host="127.0.0.1", port=8000)




    ApiClient:



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



    Models:



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



     Grid:


     <Grid>
    <TextBlock Text="Окно авторизации" FontSize="22" TextAlignment="Center" Margin="282,0,282,333" Height="39" VerticalAlignment="Bottom"></TextBlock>
    <TextBox x:Name="logintxt" Margin="282,0,251,252" BorderBrush="Black" Height="51" VerticalAlignment="Bottom"></TextBox>
    <TextBox x:Name="passwordtxt" Margin="282,0,251,166" BorderBrush="Black" Height="51" VerticalAlignment="Bottom"></TextBox>
    <Button x:Name="authorizatebutton" Content="Войти" BorderBrush="Black" Background="Black" Foreground="White" Margin="350,0,319,83" Click="authorizatebutton_Click" Height="45" VerticalAlignment="Bottom"></Button>
    <TextBlock Text="Логин:" FontSize="14" TextAlignment="Center" Margin="208,0,523,258" Height="39" VerticalAlignment="Bottom"></TextBlock>
    <TextBlock Text="Пароль:" FontSize="14" TextAlignment="Center" Margin="213,0,528,167" Height="40" VerticalAlignment="Bottom"></TextBlock>
    </Grid>



      c#:

      
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

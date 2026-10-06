#Models
Category.cs

using System;
using System.Collections.Generic;

namespace ReadApp.Models;

public partial class Category
{
    public int Idcategory { get; set; }

    public string? Category1 { get; set; }

    public virtual ICollection<Product> Products { get; set; } = new List<Product>();
}
creator.cs

using System;
using System.Collections.Generic;

namespace ReadApp.Models;

public partial class Creator
{
    public int Idcreator { get; set; }

    public string? Creator1 { get; set; }

    public virtual ICollection<Product> Products { get; set; } = new List<Product>();
}
Order.cs

using System;
using System.Collections.Generic;

namespace ReadApp.Models;

public partial class Order
{
    public int Idorders { get; set; }

    public DateOnly? Dateorder { get; set; }

    public DateOnly? Datearrive { get; set; }

    public int? Adress { get; set; }

    public int? Client { get; set; }

    public int? Code { get; set; }

    public int? Status { get; set; }

    public virtual Point? AdressNavigation { get; set; }

    public virtual User? ClientNavigation { get; set; }

    public virtual ICollection<OrderItem> OrderItems { get; set; } = new List<OrderItem>();

    public virtual Status? StatusNavigation { get; set; }
}

OrderItem.cs

using System;
using System.Collections.Generic;

namespace ReadApp.Models;

public partial class OrderItem
{
    public int IdorderItems { get; set; }

    public int? Idorder { get; set; }

    public int? Article { get; set; }

    public int? Count { get; set; }

    public virtual Product? ArticleNavigation { get; set; }

    public virtual Order? IdorderNavigation { get; set; }
}

Point.cs

using System;
using System.Collections.Generic;

namespace ReadApp.Models;

public partial class Point
{
    public int Idpoints { get; set; }

    public int? Mail { get; set; }

    public string? City { get; set; }

    public string? Street { get; set; }

    public int? Building { get; set; }

    public virtual ICollection<Order> Orders { get; set; } = new List<Order>();
}

Product.cs

using System;
using System.Collections.Generic;

namespace ReadApp.Models;

public partial class Product
{
    public int Idproducts { get; set; }

    public string? Article { get; set; }

    public string? Product1 { get; set; }

    public string? Publisher { get; set; }

    public int? Creator { get; set; }

    public int? Category { get; set; }

    public int? Percent { get; set; }

    public int? Count { get; set; }

    public string? Description { get; set; }

    public string? Photo { get; set; }

    public virtual Category? CategoryNavigation { get; set; }

    public virtual Creator? CreatorNavigation { get; set; }

    public virtual ICollection<OrderItem> OrderItems { get; set; } = new List<OrderItem>();
}


ReadBDContext.cs

using System;
using System.Collections.Generic;
using Microsoft.EntityFrameworkCore;
using Pomelo.EntityFrameworkCore.MySql.Scaffolding.Internal;

namespace ReadApp.Models;

public partial class ReadBdContext : DbContext
{
    public ReadBdContext()
    {
    }

    public ReadBdContext(DbContextOptions<ReadBdContext> options)
        : base(options)
    {
    }

    public virtual DbSet<Category> Categories { get; set; }

    public virtual DbSet<Creator> Creators { get; set; }

    public virtual DbSet<Order> Orders { get; set; }

    public virtual DbSet<OrderItem> OrderItems { get; set; }

    public virtual DbSet<Point> Points { get; set; }

    public virtual DbSet<Product> Products { get; set; }

    public virtual DbSet<Role> Roles { get; set; }

    public virtual DbSet<Status> Statuses { get; set; }

    public virtual DbSet<User> Users { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
#warning To protect potentially sensitive information in your connection string, you should move it out of source code. You can avoid scaffolding the connection string by using the Name= syntax to read it from configuration - see https://go.microsoft.com/fwlink/?linkid=2131148. For more guidance on storing connection strings, see https://go.microsoft.com/fwlink/?LinkId=723263.
        => optionsBuilder.UseMySql("server=localhost;port=3306;user=root;password=1234;database=read_bd", Microsoft.EntityFrameworkCore.ServerVersion.Parse("9.5.0-mysql"));

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder
            .UseCollation("utf8mb4_0900_ai_ci")
            .HasCharSet("utf8mb4");

        modelBuilder.Entity<Category>(entity =>
        {
            entity.HasKey(e => e.Idcategory).HasName("PRIMARY");

            entity.ToTable("categories");

            entity.Property(e => e.Idcategory)
                .ValueGeneratedNever()
                .HasColumnName("idcategory");
            entity.Property(e => e.Category1)
                .HasMaxLength(100)
                .HasColumnName("category");
        });

        modelBuilder.Entity<Creator>(entity =>
        {
            entity.HasKey(e => e.Idcreator).HasName("PRIMARY");

            entity.ToTable("creator");

            entity.Property(e => e.Idcreator)
                .ValueGeneratedNever()
                .HasColumnName("idcreator");
            entity.Property(e => e.Creator1)
                .HasMaxLength(100)
                .HasColumnName("creator");
        });

        modelBuilder.Entity<Order>(entity =>
        {
            entity.HasKey(e => e.Idorders).HasName("PRIMARY");

            entity.ToTable("orders");

            entity.HasIndex(e => e.Status, "afwegrh_idx");

            entity.HasIndex(e => e.Adress, "fasfsag_idx");

            entity.HasIndex(e => e.Client, "frewgerh_idx");

            entity.Property(e => e.Idorders)
                .ValueGeneratedNever()
                .HasColumnName("idorders");
            entity.Property(e => e.Adress).HasColumnName("adress");
            entity.Property(e => e.Client).HasColumnName("client");
            entity.Property(e => e.Code).HasColumnName("code");
            entity.Property(e => e.Datearrive).HasColumnName("datearrive");
            entity.Property(e => e.Dateorder).HasColumnName("dateorder");
            entity.Property(e => e.Status).HasColumnName("status");

            entity.HasOne(d => d.AdressNavigation).WithMany(p => p.Orders)
                .HasForeignKey(d => d.Adress)
                .HasConstraintName("fasfsag");

            entity.HasOne(d => d.ClientNavigation).WithMany(p => p.Orders)
                .HasForeignKey(d => d.Client)
                .HasConstraintName("frewgerh");

            entity.HasOne(d => d.StatusNavigation).WithMany(p => p.Orders)
                .HasForeignKey(d => d.Status)
                .HasConstraintName("afwegrh");
        });

        modelBuilder.Entity<OrderItem>(entity =>
        {
            entity.HasKey(e => e.IdorderItems).HasName("PRIMARY");

            entity.ToTable("order_items");

            entity.HasIndex(e => e.Article, "fsdgss_idx");

            entity.HasIndex(e => e.Idorder, "sdfdssg_idx");

            entity.Property(e => e.IdorderItems)
                .ValueGeneratedNever()
                .HasColumnName("idorder_items");
            entity.Property(e => e.Article).HasColumnName("article");
            entity.Property(e => e.Count).HasColumnName("count");
            entity.Property(e => e.Idorder).HasColumnName("idorder");

            entity.HasOne(d => d.ArticleNavigation).WithMany(p => p.OrderItems)
                .HasForeignKey(d => d.Article)
                .HasConstraintName("asfagd");

            entity.HasOne(d => d.IdorderNavigation).WithMany(p => p.OrderItems)
                .HasForeignKey(d => d.Idorder)
                .HasConstraintName("sdfdssg");
        });

        modelBuilder.Entity<Point>(entity =>
        {
            entity.HasKey(e => e.Idpoints).HasName("PRIMARY");

            entity.ToTable("points");

            entity.Property(e => e.Idpoints)
                .ValueGeneratedNever()
                .HasColumnName("idpoints");
            entity.Property(e => e.Building).HasColumnName("building");
            entity.Property(e => e.City)
                .HasMaxLength(45)
                .HasColumnName("city");
            entity.Property(e => e.Mail).HasColumnName("mail");
            entity.Property(e => e.Street)
                .HasMaxLength(45)
                .HasColumnName("street");
        });

        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(e => e.Idproducts).HasName("PRIMARY");

            entity.ToTable("products");

            entity.HasIndex(e => e.Creator, "SCSDGFH_idx");

            entity.HasIndex(e => e.Category, "asfadg_idx");

            entity.Property(e => e.Idproducts)
                .ValueGeneratedNever()
                .HasColumnName("idproducts");
            entity.Property(e => e.Article)
                .HasMaxLength(45)
                .HasColumnName("article");
            entity.Property(e => e.Category).HasColumnName("category");
            entity.Property(e => e.Count).HasColumnName("count");
            entity.Property(e => e.Creator).HasColumnName("creator");
            entity.Property(e => e.Description)
                .HasMaxLength(500)
                .HasColumnName("description");
            entity.Property(e => e.Percent).HasColumnName("percent");
            entity.Property(e => e.Photo)
                .HasMaxLength(45)
                .HasColumnName("photo");
            entity.Property(e => e.Product1)
                .HasMaxLength(255)
                .HasColumnName("product");
            entity.Property(e => e.Publisher)
                .HasMaxLength(100)
                .HasColumnName("publisher");

            entity.HasOne(d => d.CategoryNavigation).WithMany(p => p.Products)
                .HasForeignKey(d => d.Category)
                .HasConstraintName("asfadg");

            entity.HasOne(d => d.CreatorNavigation).WithMany(p => p.Products)
                .HasForeignKey(d => d.Creator)
                .HasConstraintName("SCSDGFH");
        });

        modelBuilder.Entity<Role>(entity =>
        {
            entity.HasKey(e => e.Idroles).HasName("PRIMARY");

            entity.ToTable("roles");

            entity.Property(e => e.Idroles)
                .ValueGeneratedNever()
                .HasColumnName("idroles");
            entity.Property(e => e.Role1)
                .HasMaxLength(200)
                .HasColumnName("role");
        });

        modelBuilder.Entity<Status>(entity =>
        {
            entity.HasKey(e => e.Idstatuses).HasName("PRIMARY");

            entity.ToTable("statuses");

            entity.Property(e => e.Idstatuses)
                .ValueGeneratedNever()
                .HasColumnName("idstatuses");
            entity.Property(e => e.Status1)
                .HasMaxLength(45)
                .HasColumnName("status");
        });

        modelBuilder.Entity<User>(entity =>
        {
            entity.HasKey(e => e.Idusers).HasName("PRIMARY");

            entity.ToTable("users");

            entity.HasIndex(e => e.Role, "dsafadg_idx");

            entity.Property(e => e.Idusers)
                .ValueGeneratedNever()
                .HasColumnName("idusers");
            entity.Property(e => e.Fname)
                .HasMaxLength(45)
                .HasColumnName("fname");
            entity.Property(e => e.Login)
                .HasMaxLength(45)
                .HasColumnName("login");
            entity.Property(e => e.Password)
                .HasMaxLength(45)
                .HasColumnName("password");
            entity.Property(e => e.Role).HasColumnName("role");
            entity.Property(e => e.Sname)
                .HasMaxLength(45)
                .HasColumnName("sname");
            entity.Property(e => e.Tname)
                .HasMaxLength(45)
                .HasColumnName("tname");

            entity.HasOne(d => d.RoleNavigation).WithMany(p => p.Users)
                .HasForeignKey(d => d.Role)
                .HasConstraintName("dsafadg");
        });

        OnModelCreatingPartial(modelBuilder);
    }

    partial void OnModelCreatingPartial(ModelBuilder modelBuilder);
}

Role.cs

using System;
using System.Collections.Generic;

namespace ReadApp.Models;

public partial class Role
{
    public int Idroles { get; set; }

    public string? Role1 { get; set; }

    public virtual ICollection<User> Users { get; set; } = new List<User>();
}

status.cs

using System;
using System.Collections.Generic;

namespace ReadApp.Models;

public partial class Role
{
    public int Idroles { get; set; }

    public string? Role1 { get; set; }

    public virtual ICollection<User> Users { get; set; } = new List<User>();
}

User.cs

using System;
using System.Collections.Generic;

namespace ReadApp.Models;

public partial class User
{
    public int Idusers { get; set; }

    public int? Role { get; set; }

    public string? Sname { get; set; }

    public string? Fname { get; set; }

    public string? Tname { get; set; }

    public string? Login { get; set; }

    public string? Password { get; set; }

    public virtual ICollection<Order> Orders { get; set; } = new List<Order>();

    public virtual Role? RoleNavigation { get; set; }
}

#App.xaml

<Application x:Class="ReadApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:local="clr-namespace:ReadApp"
             StartupUri="Auth.xaml">
    <Application.Resources>
         
    </Application.Resources>
</Application>

App.xaml.cs

using System.Configuration;
using System.Data;
using System.Windows;

namespace ReadApp
{
    /// <summary>
    /// Interaction logic for App.xaml
    /// </summary>
    public partial class App : Application
    {
    }

}

AssemblyInfo.cs

using System.Windows;

[assembly: ThemeInfo(
    ResourceDictionaryLocation.None,            //where theme specific resource dictionaries are located
                                                //(used if a resource is not found in the page,
                                                // or application resource dictionaries)
    ResourceDictionaryLocation.SourceAssembly   //where the generic resource dictionary is located
                                                //(used if a resource is not found in the page,
                                                // app, or any theme specific resource dictionaries)
)]

Auth.xaml

<Window x:Class="ReadApp.Auth"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:ReadApp"
        mc:Ignorable="d"
        Title="Auth" Height="450" Width="350">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="25*"/>
            <RowDefinition Height="58*"/>
            <RowDefinition Height="56*"/>
            <RowDefinition Height="86*"/>
            <RowDefinition Height="138*"/>
            <RowDefinition Height="71*"/>
        </Grid.RowDefinitions>

        <TextBlock Grid.Row="0" Text="Авторизация" VerticalAlignment="Center" HorizontalAlignment="Center" FontSize="20"/>

        <TextBlock Text="Логин" Margin="0,25,0,29" Grid.RowSpan="2"/>
        <TextBox x:Name="loginTB" Grid.Row="1" Margin="0,29,0,0" />

        <TextBlock Text="Пароль" Margin="0,25,0,29" Grid.RowSpan="3"/>
        <TextBox x:Name="passwordPB" Grid.Row="2" Margin="0,29,0,0" />

        <Button x:Name="Authorization" Grid.Row="3" Margin="0,54,0,0" Content="Авторизоваться" Click="Authorization_Click"/>

        <Button x:Name="Guest" Grid.Row="4" Margin="0,110,0,0" Content="Войти как гость" Click="Guest_Click"/>
    </Grid>
</Window>

GuestWindow.xaml

<Window x:Class="ReadApp.GuestWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:ReadApp"
        mc:Ignorable="d"
        Title="GuestWindow" Height="450" Width="800">
    <Grid>
        <ScrollViewer>
            <WrapPanel x:Name="panel"/>
        </ScrollViewer>
    </Grid>
</Window>


Products.xaml

<Window x:Class="ReadApp.Products"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:local="clr-namespace:ReadApp"
        mc:Ignorable="d"
        Title="Products" Height="450" Width="800">
    <Grid>
        <ScrollViewer>
            <WrapPanel x:Name="panel"/>
        </ScrollViewer>
    </Grid>
</Window>

ReadUserControl.xaml

<UserControl x:Class="ReadApp.ReadUserControl"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006" 
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008" 
             xmlns:local="clr-namespace:ReadApp"
             mc:Ignorable="d" 
             d:DesignHeight="200" d:DesignWidth="600">
    <Border Margin="3" CornerRadius="15">
    <Grid Margin="5">
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="76*"/>
            <ColumnDefinition Width="137*"/>
            <ColumnDefinition Width="87*"/>
        </Grid.ColumnDefinitions>
        
        <Border Grid.Column="0" BorderBrush="Gray" Margin="5">
            <Image x:Name="Photo"/>
        </Border>

        <Border Grid.Column="1" BorderBrush="Gray" Margin="5">
            <StackPanel>
                <TextBlock x:Name="tbTitle"/>
                <TextBlock x:Name="tbPublisher"/>
                <TextBlock x:Name="tbCreator"/>
                <TextBlock x:Name="tbCategory"/>
                <TextBlock x:Name="tbDescription"/>
                <TextBlock x:Name="tbPrice"/>
                <TextBlock x:Name="tbCount"/>
            </StackPanel>
        </Border>

        <Border Grid.Column="3" BorderBrush="Gray" Margin="5">
            <TextBlock x:Name="tbPercent"/>
        </Border>
    </Grid>
    </Border>
</UserControl>


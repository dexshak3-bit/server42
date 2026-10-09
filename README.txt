#Models
AppSettings
using System.Text.Json;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv.Models;

/// <summary>Настройки подключения к MySQL (файл appsettings.json рядом с exe).</summary>
public static class AppSettings
{
    private const string DefaultConnection =
        "server=localhost;port=3306;database=chudo_obuv;user=root;password=;CharSet=utf8mb4";

    private static string? _connectionString;
    private static ServerVersion? _serverVersion;

    public static string ConnectionString
    {
        get
        {
            if (_connectionString != null)
                return _connectionString;

            _connectionString = DefaultConnection;
            try
            {
                var file = System.IO.Path.Combine(AppContext.BaseDirectory, "appsettings.json");
                if (System.IO.File.Exists(file))
                {
                    using var doc = JsonDocument.Parse(System.IO.File.ReadAllText(file));
                    if (doc.RootElement.TryGetProperty("ConnectionString", out var value)
                        && !string.IsNullOrWhiteSpace(value.GetString()))
                    {
                        _connectionString = value.GetString()!;
                    }
                }
            }
            catch
            {
                // при ошибке чтения используется строка по умолчанию
            }

            return _connectionString;
        }
    }

    /// <summary>Версия сервера определяется один раз (MySQL или MariaDB).</summary>
    public static ServerVersion GetServerVersion()
    {
        return _serverVersion ??= ServerVersion.AutoDetect(ConnectionString);
    }
}

Category

using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Category
{
    public int CategoryId { get; set; }

    public string CategoryName { get; set; } = null!;

    public virtual ICollection<Subcategory> Subcategories { get; set; } = new List<Subcategory>();
}

ChudoObuvContext

using System;
using System.Collections.Generic;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv.Models;

public partial class ChudoObuvContext : DbContext
{
    public ChudoObuvContext()
    {
    }

    public ChudoObuvContext(DbContextOptions<ChudoObuvContext> options)
        : base(options)
    {
    }

    public virtual DbSet<Category> Categories { get; set; }

    public virtual DbSet<Manufacturer> Manufacturers { get; set; }

    public virtual DbSet<Order> Orders { get; set; }

    public virtual DbSet<OrderItem> OrderItems { get; set; }

    public virtual DbSet<Product> Products { get; set; }

    public virtual DbSet<Role> Roles { get; set; }

    public virtual DbSet<StockItem> StockItems { get; set; }

    public virtual DbSet<Subcategory> Subcategories { get; set; }

    public virtual DbSet<User> Users { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            // Строка подключения хранится в appsettings.json (рядом с exe).
            optionsBuilder
                .UseLazyLoadingProxies()
                .UseMySql(AppSettings.ConnectionString, AppSettings.GetServerVersion());
        }
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Category>(entity =>
        {
            entity.HasKey(e => e.CategoryId).HasName("PRIMARY");

            entity.ToTable("categories");

            entity.HasIndex(e => e.CategoryName, "uq_categories_name").IsUnique();

            entity.Property(e => e.CategoryId).HasColumnName("category_id");
            entity.Property(e => e.CategoryName)
                .HasMaxLength(60)
                .HasColumnName("category_name");
        });

        modelBuilder.Entity<Manufacturer>(entity =>
        {
            entity.HasKey(e => e.ManufacturerId).HasName("PRIMARY");

            entity.ToTable("manufacturers");

            entity.HasIndex(e => e.ManufacturerName, "uq_manufacturers_name").IsUnique();

            entity.Property(e => e.ManufacturerId).HasColumnName("manufacturer_id");
            entity.Property(e => e.ManufacturerName)
                .HasMaxLength(100)
                .HasColumnName("manufacturer_name");
        });

        modelBuilder.Entity<Order>(entity =>
        {
            entity.HasKey(e => e.OrderId).HasName("PRIMARY");

            entity.ToTable("orders");

            entity.HasIndex(e => e.UserId, "ix_orders_user");

            entity.Property(e => e.OrderId).HasColumnName("order_id");
            entity.Property(e => e.OrderDate).HasColumnName("order_date");
            entity.Property(e => e.UserId).HasColumnName("user_id");

            entity.HasOne(d => d.User).WithMany(p => p.Orders)
                .HasForeignKey(d => d.UserId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_orders_user");
        });

        modelBuilder.Entity<OrderItem>(entity =>
        {
            entity.HasKey(e => e.OrderItemId).HasName("PRIMARY");

            entity.ToTable("order_items", tb => tb.HasCheckConstraint("ck_oi_quantity", "quantity > 0"));

            entity.HasIndex(e => e.StockItemId, "ix_oi_stock");

            entity.HasIndex(e => new { e.OrderId, e.StockItemId }, "uq_order_stock").IsUnique();

            entity.Property(e => e.OrderItemId).HasColumnName("order_item_id");
            entity.Property(e => e.OrderId).HasColumnName("order_id");
            entity.Property(e => e.Quantity).HasColumnName("quantity");
            entity.Property(e => e.StockItemId).HasColumnName("stock_item_id");
            entity.Property(e => e.UnitPrice)
                .HasPrecision(10, 2)
                .HasColumnName("unit_price");

            entity.HasOne(d => d.Order).WithMany(p => p.OrderItems)
                .HasForeignKey(d => d.OrderId)
                .HasConstraintName("fk_oi_order");

            entity.HasOne(d => d.StockItem).WithMany(p => p.OrderItems)
                .HasForeignKey(d => d.StockItemId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_oi_stock");
        });

        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(e => e.ProductId).HasName("PRIMARY");

            entity.ToTable("products");

            entity.HasIndex(e => e.ManufacturerId, "ix_products_man");

            entity.HasIndex(e => e.SubcategoryId, "ix_products_sub");

            entity.HasIndex(e => new { e.ProductName, e.ManufacturerId }, "uq_products_name_man").IsUnique();

            entity.Property(e => e.ProductId).HasColumnName("product_id");
            entity.Property(e => e.Composition)
                .HasColumnType("text")
                .HasColumnName("composition");
            entity.Property(e => e.Description)
                .HasColumnType("text")
                .HasColumnName("description");
            entity.Property(e => e.ImageFile)
                .HasMaxLength(100)
                .HasComment("имя файла изображения в папке Images приложения")
                .HasColumnName("image_file");
            entity.Property(e => e.ManufacturerId).HasColumnName("manufacturer_id");
            entity.Property(e => e.Price)
                .HasPrecision(10, 2)
                .HasColumnName("price");
            entity.Property(e => e.ProductName)
                .HasMaxLength(200)
                .HasColumnName("product_name");
            entity.Property(e => e.SubcategoryId).HasColumnName("subcategory_id");

            entity.HasOne(d => d.Manufacturer).WithMany(p => p.Products)
                .HasForeignKey(d => d.ManufacturerId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_products_man");

            entity.HasOne(d => d.Subcategory).WithMany(p => p.Products)
                .HasForeignKey(d => d.SubcategoryId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_products_sub");
        });

        modelBuilder.Entity<Role>(entity =>
        {
            entity.HasKey(e => e.RoleId).HasName("PRIMARY");

            entity.ToTable("roles");

            entity.HasIndex(e => e.RoleName, "uq_roles_name").IsUnique();

            entity.Property(e => e.RoleId).HasColumnName("role_id");
            entity.Property(e => e.RoleName)
                .HasMaxLength(50)
                .HasColumnName("role_name");
        });

        modelBuilder.Entity<StockItem>(entity =>
        {
            entity.HasKey(e => e.StockItemId).HasName("PRIMARY");

            entity.ToTable("stock_items", tb => tb.HasCheckConstraint("ck_stock_quantity", "quantity >= 0"));

            entity.HasIndex(e => new { e.ProductId, e.SizeValue }, "uq_stock_product_size").IsUnique();

            entity.Property(e => e.StockItemId).HasColumnName("stock_item_id");
            entity.Property(e => e.ProductId).HasColumnName("product_id");
            entity.Property(e => e.Quantity).HasColumnName("quantity");
            entity.Property(e => e.SizeValue)
                .HasPrecision(3, 1)
                .HasColumnName("size_value");

            entity.HasOne(d => d.Product).WithMany(p => p.StockItems)
                .HasForeignKey(d => d.ProductId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_stock_product");

        });

        modelBuilder.Entity<Subcategory>(entity =>
        {
            entity.HasKey(e => e.SubcategoryId).HasName("PRIMARY");

            entity.ToTable("subcategories");

            entity.HasIndex(e => new { e.CategoryId, e.SubcategoryName }, "uq_subcategories").IsUnique();

            entity.Property(e => e.SubcategoryId).HasColumnName("subcategory_id");
            entity.Property(e => e.CategoryId).HasColumnName("category_id");
            entity.Property(e => e.SubcategoryName)
                .HasMaxLength(60)
                .HasColumnName("subcategory_name");

            entity.HasOne(d => d.Category).WithMany(p => p.Subcategories)
                .HasForeignKey(d => d.CategoryId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_subcategories_category");
        });

        modelBuilder.Entity<User>(entity =>
        {
            entity.HasKey(e => e.UserId).HasName("PRIMARY");

            entity.ToTable("users");

            entity.HasIndex(e => e.Login, "uq_users_login").IsUnique();

            entity.HasIndex(e => e.RoleId, "ix_users_role");

            entity.Property(e => e.UserId).HasColumnName("user_id");
            entity.Property(e => e.FirstName)
                .HasMaxLength(60)
                .HasColumnName("first_name");
            entity.Property(e => e.LastName)
                .HasMaxLength(60)
                .HasColumnName("last_name");
            entity.Property(e => e.Login)
                .HasMaxLength(50)
                .HasColumnName("login");
            entity.Property(e => e.Patronymic)
                .HasMaxLength(60)
                .HasColumnName("patronymic");
            entity.Property(e => e.RoleId).HasColumnName("role_id");

            entity.HasOne(d => d.Role).WithMany(p => p.Users)
                .HasForeignKey(d => d.RoleId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_users_role");
        });

        OnModelCreatingPartial(modelBuilder);
    }

    partial void OnModelCreatingPartial(ModelBuilder modelBuilder);
}

Manufacturer

using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Manufacturer
{
    public int ManufacturerId { get; set; }

    public string ManufacturerName { get; set; } = null!;

    public virtual ICollection<Product> Products { get; set; } = new List<Product>();
}

Order

using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Order
{
    public int OrderId { get; set; }

    public DateOnly OrderDate { get; set; }

    public int UserId { get; set; }

    public virtual ICollection<OrderItem> OrderItems { get; set; } = new List<OrderItem>();

    public virtual User User { get; set; } = null!;
}


OrderItem

using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class OrderItem
{
    public int OrderItemId { get; set; }

    public int OrderId { get; set; }

    public int StockItemId { get; set; }

    public int Quantity { get; set; }

    public decimal UnitPrice { get; set; }

    public virtual Order Order { get; set; } = null!;

    public virtual StockItem StockItem { get; set; } = null!;
}

PartialModels

using System.ComponentModel.DataAnnotations.Schema;

namespace ChudoObuv.Models;

// Дополнения к классам, созданным Scaffold-DbContext.
// Лежат в отдельном файле, поэтому не затираются при повторном scaffold (-Force).

public partial class User
{
    [NotMapped]
    public string Fio => string.Join(" ",
        new[] { LastName, FirstName, Patronymic }.Where(s => !string.IsNullOrWhiteSpace(s)));
}

public partial class Product
{
    public const int LowStockLimit = 3;

    [NotMapped]
    public string CategoryText => $"{Subcategory.Category.CategoryName} / {Subcategory.SubcategoryName}";

    [NotMapped]
    public string CategoryName => Subcategory.Category.CategoryName;

    /// <summary>Доступное для заказа количество суммарно по всем размерам модели.</summary>
    [NotMapped]
    public int TotalQuantity => StockItems.Sum(s => s.Quantity);

    /// <summary>Товары с суммарным количеством &lt;= 3 подсвечиваются в списке.</summary>
    [NotMapped]
    public bool IsLowStock => TotalQuantity <= LowStockLimit;

    /// <summary>Размеры, доступные для заказа (количество &gt; 0), по возрастанию.</summary>
    [NotMapped]
    public List<StockItem> AvailableStock => StockItems
        .Where(s => s.Quantity > 0)
        .OrderBy(s => s.SizeValue)
        .ToList();

    [NotMapped]
    public string SizesText
    {
        get
        {
            var list = AvailableStock;
            return list.Count == 0
                ? "нет в наличии"
                : string.Join(", ", list.Select(s => s.SizeText));
        }
    }

    [NotMapped]
    public string ImagePath => System.IO.Path.Combine(AppContext.BaseDirectory, "Images",
        string.IsNullOrWhiteSpace(ImageFile) ? "picture.png" : ImageFile);

    [NotMapped]
    public string PriceText => Price.ToString("N0") + " ₽";
}

public partial class StockItem
{
    [NotMapped]
    public string SizeText => SizeValue.ToString("0.#");

    [NotMapped]
    public string SizeDisplay => $"Размер {SizeText} (в наличии: {Quantity})";

    [NotMapped]
    public string FullDisplay => $"{Product.ProductName} — размер {SizeText} (в наличии: {Quantity})";
}

public partial class Order
{
    [NotMapped]
    public string CustomerFio => User.Fio;

    [NotMapped]
    public decimal Total => OrderItems.Sum(i => i.Quantity * i.UnitPrice);

    [NotMapped]
    public int ItemsCount => OrderItems.Sum(i => i.Quantity);
}

public partial class OrderItem
{
    [NotMapped]
    public string ProductName => StockItem.Product.ProductName;

    [NotMapped]
    public string SizeText => StockItem.SizeText;

    [NotMapped]
    public decimal Sum => Quantity * UnitPrice;
}

Product

using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Product
{
    public int ProductId { get; set; }

    public string ProductName { get; set; } = null!;

    public int ManufacturerId { get; set; }

    public int SubcategoryId { get; set; }

    public string Description { get; set; } = null!;

    public string Composition { get; set; } = null!;

    /// <summary>имя файла изображения в папке Images приложения</summary>
    public string? ImageFile { get; set; }

    public decimal Price { get; set; }

    public virtual Manufacturer Manufacturer { get; set; } = null!;

    public virtual ICollection<StockItem> StockItems { get; set; } = new List<StockItem>();

    public virtual Subcategory Subcategory { get; set; } = null!;
}


Role

using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Role
{
    public int RoleId { get; set; }

    public string RoleName { get; set; } = null!;

    public virtual ICollection<User> Users { get; set; } = new List<User>();
}


StockItem

using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class StockItem
{
    public int StockItemId { get; set; }

    public int ProductId { get; set; }

    public decimal SizeValue { get; set; }

    public int Quantity { get; set; }

    public virtual ICollection<OrderItem> OrderItems { get; set; } = new List<OrderItem>();

    public virtual Product Product { get; set; } = null!;
}


Subcategory

using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Subcategory
{
    public int SubcategoryId { get; set; }

    public int CategoryId { get; set; }

    public string SubcategoryName { get; set; } = null!;

    public virtual Category Category { get; set; } = null!;

    public virtual ICollection<Product> Products { get; set; } = new List<Product>();
}


User

using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class User
{
    public int UserId { get; set; }

    public string LastName { get; set; } = null!;

    public string FirstName { get; set; } = null!;

    public string? Patronymic { get; set; }

    public string Login { get; set; } = null!;

    public int RoleId { get; set; }

    public virtual ICollection<Order> Orders { get; set; } = new List<Order>();

    public virtual Role Role { get; set; } = null!;
}


cart.services

using System.Collections.ObjectModel;
using System.ComponentModel;
using ChudoObuv.Models;

namespace ChudoObuv.Services;

/// <summary>Строка формируемого заказа.</summary>
public class CartLine : INotifyPropertyChanged
{
    private int _quantity;

    public CartLine(StockItem item, int quantity)
    {
        Item = item;
        _quantity = quantity;
    }

    public StockItem Item { get; }

    public string ProductName => Item.Product.ProductName;
    public string SizeText => Item.SizeText;
    public decimal UnitPrice => Item.Product.Price;
    public decimal Sum => Quantity * UnitPrice;

    public int Quantity
    {
        get => _quantity;
        set
        {
            _quantity = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Quantity)));
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Sum)));
        }
    }

    public event PropertyChangedEventHandler? PropertyChanged;
}

/// <summary>Корзина (формируемый заказ) текущего пользователя.</summary>
public static class Cart
{
    public static ObservableCollection<CartLine> Lines { get; } = new();

    public static event Action? Changed;

    public static int TotalCount => Lines.Sum(l => l.Quantity);

    public static decimal TotalSum => Lines.Sum(l => l.Sum);

    /// <summary>Добавляет одну пару выбранной товарной позиции.</summary>
    public static bool TryAdd(StockItem item, out string message)
    {
        var line = Lines.FirstOrDefault(l => l.Item.StockItemId == item.StockItemId);
        int newQty = (line?.Quantity ?? 0) + 1;
        if (newQty > item.Quantity)
        {
            message = $"Доступно для заказа только {item.Quantity} пар.";
            return false;
        }

        if (line == null)
            Lines.Add(new CartLine(item, 1));
        else
            line.Quantity = newQty;

        message = "";
        RaiseChanged();
        return true;
    }

    public static bool TryChangeQuantity(CartLine line, int delta, out string message)
    {
        int newQty = line.Quantity + delta;
        message = "";
        if (newQty < 1)
            return false;
        if (newQty > line.Item.Quantity)
        {
            message = $"Доступно для заказа только {line.Item.Quantity} пар.";
            return false;
        }

        line.Quantity = newQty;
        RaiseChanged();
        return true;
    }

    public static void Remove(CartLine line)
    {
        Lines.Remove(line);
        RaiseChanged();
    }

    public static void Clear()
    {
        Lines.Clear();
        RaiseChanged();
    }

    public static void RaiseChanged() => Changed?.Invoke();
}


session.services

using ChudoObuv.Models;

namespace ChudoObuv.Services;

public enum AppRole
{
    Guest,       // неавторизованный пользователь
    User,        // авторизованный пользователь
    Manager,     // менеджер
    Admin        // администратор
}

/// <summary>Текущий пользователь приложения.</summary>
public static class Session
{
    public static int? UserId { get; private set; }
    public static string Fio { get; private set; } = "Гость";
    public static string RoleName { get; private set; } = "Неавторизованный пользователь";
    public static AppRole Role { get; private set; } = AppRole.Guest;

    /// <summary>Оформлять заказы может любой авторизованный пользователь.</summary>
    public static bool CanOrder => Role != AppRole.Guest;

    /// <summary>Поиск, фильтрация, сортировка каталога — только авторизованным.</summary>
    public static bool CanFilter => Role != AppRole.Guest;

    /// <summary>Просмотр списка заказов, добавление и удаление — менеджер и администратор.</summary>
    public static bool CanManageOrders => Role == AppRole.Manager || Role == AppRole.Admin;

    /// <summary>Редактирование заказа — только администратор.</summary>
    public static bool CanEditOrders => Role == AppRole.Admin;

    public static void Login(User user)
    {
        UserId = user.UserId;
        Fio = user.Fio;
        RoleName = user.Role.RoleName;
        Role = RoleName switch
        {
            "Администратор" => AppRole.Admin,
            "Менеджер" => AppRole.Manager,
            _ => AppRole.User
        };
    }

    public static void Logout()
    {
        UserId = null;
        Fio = "Гость";
        RoleName = "Неавторизованный пользователь";
        Role = AppRole.Guest;
    }
}

App

<Application x:Class="ChudoObuv.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             StartupUri="Auth.xaml">
    <Application.Resources>

        <!-- ===== Цвета по руководству по стилю ===== -->
        <SolidColorBrush x:Key="MainBg" Color="#FFFFFF"/>
        <SolidColorBrush x:Key="AltBg" Color="#D2F6E7"/>
        <SolidColorBrush x:Key="Accent" Color="#70B2AF"/>
        <SolidColorBrush x:Key="AccentDark" Color="#3E7C79"/>
        <SolidColorBrush x:Key="LowStockBg" Color="#FF8080"/>
        <SolidColorBrush x:Key="TextBrush" Color="#1F2D2B"/>
        <SolidColorBrush x:Key="MutedBrush" Color="#4F6B68"/>
        <SolidColorBrush x:Key="ErrorBrush" Color="#C0392B"/>

        <!-- Единый радиус скругления -->
        <CornerRadius x:Key="Radius">2</CornerRadius>

        <FontFamily x:Key="AppFont">Calibri</FontFamily>

        <!-- ===== Окна ===== -->
        <Style x:Key="AppWindow" TargetType="Window">
            <Setter Property="FontFamily" Value="{StaticResource AppFont}"/>
            <Setter Property="FontSize" Value="14"/>
            <Setter Property="Background" Value="{StaticResource MainBg}"/>
            <Setter Property="Foreground" Value="{StaticResource TextBrush}"/>
        </Style>

        <!-- Заголовок формы -->
        <Style x:Key="Heading" TargetType="TextBlock">
            <Setter Property="FontSize" Value="24"/>
            <Setter Property="FontWeight" Value="Bold"/>
            <Setter Property="Foreground" Value="{StaticResource TextBrush}"/>
            <Setter Property="Margin" Value="0,0,0,10"/>
        </Style>

        <!-- Подпись над полем -->
        <Style x:Key="FieldLabel" TargetType="TextBlock">
            <Setter Property="FontWeight" Value="SemiBold"/>
            <Setter Property="Foreground" Value="{StaticResource MutedBrush}"/>
            <Setter Property="Margin" Value="0,0,0,3"/>
        </Style>

        <!-- Карточка / панель -->
        <Style x:Key="Card" TargetType="Border">
            <Setter Property="Background" Value="{StaticResource MainBg}"/>
            <Setter Property="BorderBrush" Value="{StaticResource Accent}"/>
            <Setter Property="BorderThickness" Value="1"/>
            <Setter Property="CornerRadius" Value="{StaticResource Radius}"/>
            <Setter Property="Padding" Value="12"/>
        </Style>

        <Style x:Key="Panel" TargetType="Border">
            <Setter Property="Background" Value="{StaticResource AltBg}"/>
            <Setter Property="CornerRadius" Value="{StaticResource Radius}"/>
            <Setter Property="Padding" Value="12"/>
        </Style>

        <!-- ===== Кнопки ===== -->
        <Style TargetType="Button">
            <Setter Property="Background" Value="{StaticResource AltBg}"/>
            <Setter Property="BorderBrush" Value="{StaticResource Accent}"/>
            <Setter Property="Foreground" Value="{StaticResource TextBrush}"/>
            <Setter Property="BorderThickness" Value="1"/>
            <Setter Property="Padding" Value="16,7"/>
            <Setter Property="Margin" Value="4"/>
            <Setter Property="Cursor" Value="Hand"/>
            <Setter Property="FocusVisualStyle" Value="{x:Null}"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="Button">
                        <Border x:Name="Bd"
                                Background="{TemplateBinding Background}"
                                BorderBrush="{TemplateBinding BorderBrush}"
                                BorderThickness="{TemplateBinding BorderThickness}"
                                CornerRadius="{StaticResource Radius}"
                                Padding="{TemplateBinding Padding}">
                            <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsMouseOver" Value="True">
                                <Setter TargetName="Bd" Property="BorderBrush" Value="{StaticResource AccentDark}"/>
                                <Setter TargetName="Bd" Property="Opacity" Value="0.85"/>
                            </Trigger>
                            <Trigger Property="IsPressed" Value="True">
                                <Setter TargetName="Bd" Property="Opacity" Value="0.7"/>
                            </Trigger>
                            <Trigger Property="IsEnabled" Value="False">
                                <Setter TargetName="Bd" Property="Opacity" Value="0.4"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>

        <!-- Кнопка целевого действия: акцентный цвет -->
        <Style x:Key="AccentButton" TargetType="Button" BasedOn="{StaticResource {x:Type Button}}">
            <Setter Property="Background" Value="{StaticResource Accent}"/>
            <Setter Property="BorderBrush" Value="{StaticResource Accent}"/>
            <Setter Property="FontWeight" Value="SemiBold"/>
        </Style>

        <!-- Белая кнопка для шапки -->
        <Style x:Key="LightButton" TargetType="Button" BasedOn="{StaticResource {x:Type Button}}">
            <Setter Property="Background" Value="{StaticResource MainBg}"/>
        </Style>

        <!-- ===== Поля ввода ===== -->
        <Style TargetType="TextBox">
            <Setter Property="Padding" Value="8,5"/>
            <Setter Property="Background" Value="{StaticResource MainBg}"/>
            <Setter Property="BorderBrush" Value="{StaticResource Accent}"/>
            <Setter Property="BorderThickness" Value="1"/>
            <Setter Property="VerticalContentAlignment" Value="Center"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="TextBox">
                        <Border x:Name="Bd"
                                Background="{TemplateBinding Background}"
                                BorderBrush="{TemplateBinding BorderBrush}"
                                BorderThickness="{TemplateBinding BorderThickness}"
                                CornerRadius="{StaticResource Radius}">
                            <ScrollViewer x:Name="PART_ContentHost" Margin="{TemplateBinding Padding}"
                                          VerticalAlignment="Center"/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsKeyboardFocused" Value="True">
                                <Setter TargetName="Bd" Property="BorderBrush" Value="{StaticResource AccentDark}"/>
                            </Trigger>
                            <Trigger Property="IsEnabled" Value="False">
                                <Setter TargetName="Bd" Property="Opacity" Value="0.5"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>

        <!-- ComboBox: кнопка-стрелка -->
        <ControlTemplate x:Key="ComboToggleTemplate" TargetType="ToggleButton">
            <Border x:Name="Bd"
                    Background="{TemplateBinding Background}"
                    BorderBrush="{TemplateBinding BorderBrush}"
                    BorderThickness="{TemplateBinding BorderThickness}"
                    CornerRadius="{StaticResource Radius}">
                <Path HorizontalAlignment="Right" VerticalAlignment="Center" Margin="0,0,10,0"
                      Data="M 0 0 L 4 4 L 8 0 Z" Fill="{StaticResource AccentDark}"/>
            </Border>
            <ControlTemplate.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter TargetName="Bd" Property="BorderBrush" Value="{StaticResource AccentDark}"/>
                </Trigger>
            </ControlTemplate.Triggers>
        </ControlTemplate>

        <Style TargetType="ComboBoxItem">
            <Setter Property="Padding" Value="8,5"/>
            <Setter Property="Foreground" Value="{StaticResource TextBrush}"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="ComboBoxItem">
                        <Border x:Name="Bd" Background="Transparent" Padding="{TemplateBinding Padding}"
                                CornerRadius="{StaticResource Radius}">
                            <ContentPresenter/>
                        </Border>
                        <ControlTemplate.Triggers>
                            <Trigger Property="IsHighlighted" Value="True">
                                <Setter TargetName="Bd" Property="Background" Value="{StaticResource AltBg}"/>
                            </Trigger>
                            <Trigger Property="IsSelected" Value="True">
                                <Setter TargetName="Bd" Property="Background" Value="{StaticResource AltBg}"/>
                            </Trigger>
                        </ControlTemplate.Triggers>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>

        <Style TargetType="ComboBox">
            <Setter Property="Background" Value="{StaticResource MainBg}"/>
            <Setter Property="BorderBrush" Value="{StaticResource Accent}"/>
            <Setter Property="BorderThickness" Value="1"/>
            <Setter Property="Padding" Value="8,5"/>
            <Setter Property="MinHeight" Value="32"/>
            <Setter Property="FocusVisualStyle" Value="{x:Null}"/>
            <Setter Property="Template">
                <Setter.Value>
                    <ControlTemplate TargetType="ComboBox">
                        <Grid>
                            <ToggleButton Focusable="False"
                                          Template="{StaticResource ComboToggleTemplate}"
                                          Background="{TemplateBinding Background}"
                                          BorderBrush="{TemplateBinding BorderBrush}"
                                          BorderThickness="{TemplateBinding BorderThickness}"
                                          ClickMode="Press"
                                          IsChecked="{Binding IsDropDownOpen, Mode=TwoWay, RelativeSource={RelativeSource TemplatedParent}}"/>
                            <ContentPresenter IsHitTestVisible="False"
                                              Content="{TemplateBinding SelectionBoxItem}"
                                              ContentTemplate="{TemplateBinding SelectionBoxItemTemplate}"
                                              ContentTemplateSelector="{TemplateBinding ItemTemplateSelector}"
                                              Margin="{TemplateBinding Padding}"
                                              VerticalAlignment="Center" HorizontalAlignment="Left"/>
                            <Popup x:Name="PART_Popup" Placement="Bottom" Focusable="False"
                                   AllowsTransparency="True"
                                   IsOpen="{TemplateBinding IsDropDownOpen}"
                                   PopupAnimation="Fade">
                                <Grid MinWidth="{TemplateBinding ActualWidth}"
                                      MaxHeight="{TemplateBinding MaxDropDownHeight}">
                                    <Border Background="{StaticResource MainBg}"
                                            BorderBrush="{StaticResource Accent}"
                                            BorderThickness="1"
                                            CornerRadius="{StaticResource Radius}"
                                            Margin="0,2,0,0"/>
                                    <ScrollViewer Margin="3,5" SnapsToDevicePixels="True">
                                        <StackPanel IsItemsHost="True"
                                                    KeyboardNavigation.DirectionalNavigation="Contained"/>
                                    </ScrollViewer>
                                </Grid>
                            </Popup>
                        </Grid>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
        </Style>

        <!-- ===== Таблицы ===== -->
        <Style TargetType="DataGridColumnHeader">
            <Setter Property="Background" Value="{StaticResource AltBg}"/>
            <Setter Property="Foreground" Value="{StaticResource TextBrush}"/>
            <Setter Property="FontWeight" Value="SemiBold"/>
            <Setter Property="Padding" Value="8,7"/>
            <Setter Property="BorderBrush" Value="{StaticResource Accent}"/>
            <Setter Property="BorderThickness" Value="0,0,1,1"/>
        </Style>

        <Style TargetType="DataGridCell">
            <Setter Property="Padding" Value="6,5"/>
            <Setter Property="BorderThickness" Value="0"/>
            <Setter Property="FocusVisualStyle" Value="{x:Null}"/>
            <Style.Triggers>
                <Trigger Property="IsSelected" Value="True">
                    <Setter Property="Background" Value="Transparent"/>
                    <Setter Property="BorderBrush" Value="Transparent"/>
                    <Setter Property="Foreground" Value="{StaticResource TextBrush}"/>
                </Trigger>
            </Style.Triggers>
        </Style>

        <Style TargetType="DataGridRow">
            <Setter Property="Background" Value="{StaticResource MainBg}"/>
            <Setter Property="Foreground" Value="{StaticResource TextBrush}"/>
            <Style.Triggers>
                <Trigger Property="IsSelected" Value="True">
                    <Setter Property="Background" Value="{StaticResource AltBg}"/>
                    <Setter Property="Foreground" Value="{StaticResource TextBrush}"/>
                </Trigger>
            </Style.Triggers>
        </Style>

        <Style TargetType="DataGrid">
            <Setter Property="BorderBrush" Value="{StaticResource Accent}"/>
            <Setter Property="Background" Value="{StaticResource MainBg}"/>
            <Setter Property="GridLinesVisibility" Value="Horizontal"/>
            <Setter Property="HorizontalGridLinesBrush" Value="{StaticResource AltBg}"/>
            <Setter Property="HeadersVisibility" Value="Column"/>
            <Setter Property="RowHeaderWidth" Value="0"/>
        </Style>

    </Application.Resources>
</Application>

App.cs

using System.Globalization;
using System.Windows;
using System.Windows.Markup;
using ChudoObuv.Models;

namespace ChudoObuv
{
    public partial class App : Application
    {
        private static ChudoObuvContext? _db;

        /// <summary>Единый контекст БД на время сеанса пользователя
        /// (нужен для ленивой загрузки связанных данных через Proxies и для общей корзины).</summary>
        public static ChudoObuvContext Db => _db ??= new ChudoObuvContext();

        public static void ResetDb()
        {
            _db?.Dispose();
            _db = null;
        }

        public App()
        {
            var ru = new CultureInfo("ru-RU");
            CultureInfo.DefaultThreadCurrentCulture = ru;
            CultureInfo.DefaultThreadCurrentUICulture = ru;
            CultureInfo.CurrentCulture = ru;
            CultureInfo.CurrentUICulture = ru;
            FrameworkElement.LanguageProperty.OverrideMetadata(
                typeof(FrameworkElement),
                new FrameworkPropertyMetadata(XmlLanguage.GetLanguage(ru.IetfLanguageTag)));
        }
    }
}

appsetings

{
  "ConnectionString": "server=localhost;port=3306;database=chudo_obuv;user=root;password=;CharSet=utf8mb4"
}

auth

<Window x:Class="ChudoObuv.Auth"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Style="{StaticResource AppWindow}"
        Title="Чудо Обувь — Авторизация"
        Icon="/Resources/app.ico"
        Height="580" Width="420"
        WindowStartupLocation="CenterScreen"
        ResizeMode="NoResize"
        Background="{StaticResource AltBg}">
    <Grid>
        <Border Style="{StaticResource Card}" Margin="24" Padding="28,20">
            <Grid>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                </Grid.RowDefinitions>

                <Image Grid.Row="0" Source="/Resources/logo.png" Height="150" Stretch="Uniform"/>

                <TextBlock Grid.Row="1" Text="Чудо Обувь" Style="{StaticResource Heading}"
                           HorizontalAlignment="Center" Margin="0,0,0,0"/>
                <TextBlock Grid.Row="2" Text="Авторизация" FontSize="16" Foreground="{StaticResource MutedBrush}"
                           HorizontalAlignment="Center" Margin="0,2,0,18"/>

                <StackPanel Grid.Row="3">
                    <TextBlock Text="Логин" Style="{StaticResource FieldLabel}"/>
                    <TextBox x:Name="loginTB" FontSize="16" Height="38"/>
                    <TextBlock x:Name="ErrorText" Foreground="{StaticResource ErrorBrush}" TextWrapping="Wrap"
                               Margin="0,8,0,0" Visibility="Collapsed"/>
                </StackPanel>

                <StackPanel Grid.Row="4" Margin="0,18,0,0">
                    <Button x:Name="Authorization" Content="Войти" Style="{StaticResource AccentButton}"
                            IsDefault="True" Margin="0" Height="40" FontSize="16" Click="Authorization_Click"/>
                    <Button x:Name="Guest" Content="Войти как гость" Margin="0,8,0,0" Height="36"
                            Click="Guest_Click"/>
                </StackPanel>
            </Grid>
        </Border>
    </Grid>
</Window>

auth.cs

using System.Windows;
using ChudoObuv.Services;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv
{
    /// <summary>Авторизация: вход только по логину, пароль не требуется.</summary>
    public partial class Auth : Window
    {
        public Auth()
        {
            InitializeComponent();
            Loaded += (_, _) => loginTB.Focus();
        }

        private void Authorization_Click(object sender, RoutedEventArgs e)
        {
            string login = loginTB.Text.Trim();
            if (login.Length == 0)
            {
                ShowError("Введите логин.");
                return;
            }

            try
            {
                var user = App.Db.Users
                    .Include(u => u.Role)
                    .FirstOrDefault(u => u.Login == login);

                if (user == null)
                {
                    ShowError("Пользователь с таким логином не найден.");
                    return;
                }

                Session.Login(user);
                new Products(user).Show();
                Close();
            }
            catch (Exception ex)
            {
                DbError(ex);
            }
        }

        private void Guest_Click(object sender, RoutedEventArgs e)
        {
            Session.Logout();
            try
            {
                App.Db.Products.Any();      // проверяем, что БД доступна
                new GuestWindow().Show();
                Close();
            }
            catch (Exception ex)
            {
                DbError(ex);
            }
        }

        private void DbError(Exception ex)
        {
            App.ResetDb();
            ShowError("Не удалось подключиться к базе данных. Проверьте строку подключения в appsettings.json "
                      + "и запущен ли сервер MySQL.\n" + ex.GetBaseException().Message);
        }

        private void ShowError(string text)
        {
            ErrorText.Text = text;
            ErrorText.Visibility = Visibility.Visible;
        }
    }
}

cartWindow

<Window x:Class="ChudoObuv.CartWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Style="{StaticResource AppWindow}"
        Title="Чудо Обувь — Оформление заказа"
        Icon="/Resources/app.ico"
        Width="860" Height="560" MinWidth="700" MinHeight="420"
        WindowStartupLocation="CenterOwner">
    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <TextBlock Grid.Row="0" Text="Оформление заказа" Style="{StaticResource Heading}"/>

        <Grid Grid.Row="1" Margin="0,0,0,10">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="Auto"/>
                <ColumnDefinition Width="*"/>
            </Grid.ColumnDefinitions>
            <TextBlock Grid.Column="0" Text="Клиент:" FontWeight="SemiBold" VerticalAlignment="Center" Margin="0,0,8,0"/>
            <TextBlock x:Name="CustomerText" Grid.Column="1" VerticalAlignment="Center"/>
            <ComboBox x:Name="CustomerBox" Grid.Column="1" DisplayMemberPath="Fio" Visibility="Collapsed"/>
        </Grid>

        <DataGrid x:Name="LinesGrid" Grid.Row="2" AutoGenerateColumns="False" CanUserAddRows="False"
                  IsReadOnly="True" SelectionMode="Single" CanUserSortColumns="False">
            <DataGrid.Columns>
                <DataGridTextColumn Header="Товар" Binding="{Binding ProductName}" Width="*"/>
                <DataGridTextColumn Header="Размер" Binding="{Binding SizeText}" Width="80"/>
                <DataGridTextColumn Header="Цена" Binding="{Binding UnitPrice, StringFormat={}{0:N0} ₽}" Width="100"/>
                <DataGridTemplateColumn Header="Количество" Width="150">
                    <DataGridTemplateColumn.CellTemplate>
                        <DataTemplate>
                            <StackPanel Orientation="Horizontal">
                                <Button Content="−" Padding="9,1" Margin="2" Click="Minus_Click"/>
                                <TextBlock Text="{Binding Quantity}" Width="32" TextAlignment="Center"
                                           VerticalAlignment="Center" FontWeight="SemiBold"/>
                                <Button Content="+" Padding="9,1" Margin="2" Click="Plus_Click"/>
                            </StackPanel>
                        </DataTemplate>
                    </DataGridTemplateColumn.CellTemplate>
                </DataGridTemplateColumn>
                <DataGridTextColumn Header="Сумма" Binding="{Binding Sum, StringFormat={}{0:N0} ₽}" Width="110"/>
                <DataGridTemplateColumn Header="" Width="Auto">
                    <DataGridTemplateColumn.CellTemplate>
                        <DataTemplate>
                            <Button Content="Удалить" Padding="8,1" Margin="2" Click="Remove_Click"/>
                        </DataTemplate>
                    </DataGridTemplateColumn.CellTemplate>
                </DataGridTemplateColumn>
            </DataGrid.Columns>
        </DataGrid>

        <DockPanel Grid.Row="3" Margin="0,12,0,0" LastChildFill="False">
            <TextBlock x:Name="TotalText" DockPanel.Dock="Left" FontSize="20" FontWeight="Bold"
                       VerticalAlignment="Center"/>
            <Button DockPanel.Dock="Right" Content="Закрыть" Click="Close_Click"/>
            <Button DockPanel.Dock="Right" x:Name="ConfirmButton" Content="Оформить заказ"
                    Style="{StaticResource AccentButton}" Click="Confirm_Click"/>
            <Button DockPanel.Dock="Right" Content="Очистить" Click="Clear_Click"/>
        </DockPanel>
    </Grid>
</Window>

cartwindow.cs

using System.Windows;
using System.Windows.Controls;
using ChudoObuv.Models;
using ChudoObuv.Services;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv;

public partial class CartWindow : Window
{
    public bool OrderCreated { get; private set; }

    public CartWindow()
    {
        InitializeComponent();
        LinesGrid.ItemsSource = Cart.Lines;
        Cart.Changed += UpdateTotal;
        Closed += (_, _) => Cart.Changed -= UpdateTotal;

        if (Session.CanManageOrders)
        {
            // менеджер и администратор оформляют заказ на выбранного клиента
            CustomerText.Visibility = Visibility.Collapsed;
            CustomerBox.Visibility = Visibility.Visible;
            var users = App.Db.Users.OrderBy(u => u.LastName).ThenBy(u => u.FirstName).ToList();
            CustomerBox.ItemsSource = users;
            CustomerBox.SelectedItem = users.FirstOrDefault(u => u.UserId == Session.UserId);
        }
        else
        {
            CustomerText.Text = Session.Fio;
        }

        UpdateTotal();
    }

    private void UpdateTotal()
    {
        TotalText.Text = $"Итого: {Cart.TotalSum:N0} ₽  ({Cart.TotalCount} пар)";
        ConfirmButton.IsEnabled = Cart.Lines.Count > 0;
    }

    private void Plus_Click(object sender, RoutedEventArgs e)
    {
        if (((FrameworkElement)sender).DataContext is CartLine line &&
            !Cart.TryChangeQuantity(line, +1, out var message) && message.Length > 0)
        {
            MessageBox.Show(message, "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Warning);
        }
    }

    private void Minus_Click(object sender, RoutedEventArgs e)
    {
        if (((FrameworkElement)sender).DataContext is CartLine line)
            Cart.TryChangeQuantity(line, -1, out _);
    }

    private void Remove_Click(object sender, RoutedEventArgs e)
    {
        if (((FrameworkElement)sender).DataContext is CartLine line)
            Cart.Remove(line);
    }

    private void Clear_Click(object sender, RoutedEventArgs e) => Cart.Clear();

    private void Close_Click(object sender, RoutedEventArgs e) => Close();

    private void Confirm_Click(object sender, RoutedEventArgs e)
    {
        if (Cart.Lines.Count == 0)
            return;

        User? customer;
        if (Session.CanManageOrders)
        {
            customer = CustomerBox.SelectedItem as User;
            if (customer == null)
            {
                MessageBox.Show("Выберите клиента.", "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Information);
                return;
            }
        }
        else
        {
            customer = App.Db.Users.Find(Session.UserId);
            if (customer == null)
                return;
        }

        try
        {
            var db = App.Db;

            // актуализируем остатки перед оформлением
            foreach (var line in Cart.Lines)
                db.Entry(line.Item).Reload();

            foreach (var line in Cart.Lines)
            {
                if (line.Quantity > line.Item.Quantity)
                {
                    MessageBox.Show(
                        $"«{line.ProductName}», размер {line.SizeText}: доступно только {line.Item.Quantity} пар.",
                        "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Warning);
                    return;
                }
            }

            var order = new Order
            {
                OrderDate = DateOnly.FromDateTime(DateTime.Today),
                UserId = customer.UserId,
                User = customer
            };

            foreach (var line in Cart.Lines)
            {
                order.OrderItems.Add(new OrderItem
                {
                    StockItemId = line.Item.StockItemId,
                    StockItem = line.Item,
                    Quantity = line.Quantity,
                    UnitPrice = line.UnitPrice
                });
                line.Item.Quantity -= line.Quantity;   // списываем со склада
            }

            db.Orders.Add(order);
            db.SaveChanges();

            MessageBox.Show($"Заказ №{order.OrderId} оформлен.\nСумма заказа: {order.Total:N0} ₽",
                "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Information);

            OrderCreated = true;
            Cart.Clear();
            Close();
        }
        catch (Exception ex)
        {
            MessageBox.Show("Не удалось оформить заказ:\n" + ex.GetBaseException().Message,
                "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Error);
        }
    }
}

GuestWindow

<Window x:Class="ChudoObuv.GuestWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Style="{StaticResource AppWindow}"
        Title="Чудо Обувь — Каталог товаров (гость)"
        Icon="/Resources/app.ico"
        Height="780" Width="1280" MinWidth="700" MinHeight="500"
        WindowStartupLocation="CenterScreen">
    <DockPanel>
        <!-- Шапка с логотипом -->
        <Border DockPanel.Dock="Top" Background="{StaticResource AltBg}"
                BorderBrush="{StaticResource Accent}" BorderThickness="0,0,0,1" Padding="14,6">
            <DockPanel LastChildFill="False">
                <Image DockPanel.Dock="Left" Source="/Resources/logo.png" Height="64" Stretch="Uniform"/>
                <StackPanel DockPanel.Dock="Left" VerticalAlignment="Center" Margin="10,0,0,0">
                    <TextBlock Text="Чудо Обувь" FontSize="26" FontWeight="Bold"/>
                    <TextBlock Text="Каталог товаров" Foreground="{StaticResource MutedBrush}"/>
                </StackPanel>
                <Button DockPanel.Dock="Right" Content="Войти" Style="{StaticResource AccentButton}"
                        VerticalAlignment="Center" Click="Login_Click"/>
                <TextBlock DockPanel.Dock="Right" Text="Вы вошли как гость" VerticalAlignment="Center"
                           Foreground="{StaticResource MutedBrush}" Margin="0,0,10,0"/>
            </DockPanel>
        </Border>

        <ScrollViewer VerticalScrollBarVisibility="Auto">
            <WrapPanel x:Name="panel" Margin="10"/>
        </ScrollViewer>
    </DockPanel>
</Window>

GuestWindow.cs

using System.Windows;
using ChudoObuv.Models;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv
{
    /// <summary>Каталог для неавторизованного пользователя (без поиска, фильтров и сортировки).</summary>
    public partial class GuestWindow : Window
    {
        public GuestWindow()
        {
            InitializeComponent();
            LoadData();
        }

        public void LoadData()
        {
            try
            {
                var allproducts = App.Db.Products
                    .Include(x => x.Manufacturer)
                    .Include(x => x.Subcategory).ThenInclude(s => s.Category)
                    .Include(x => x.StockItems)
                    .OrderBy(x => x.ProductId)
                    .ToList();

                foreach (var item in allproducts)
                {
                    var control = new ProductUserControl();
                    control.LoadData(item);
                    panel.Children.Add(control);
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show("Не удалось загрузить товары:\n" + ex.GetBaseException().Message,
                    "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Error);
            }
        }

        private void Login_Click(object sender, RoutedEventArgs e)
        {
            new Auth().Show();
            Close();
        }
    }
}

OrderEdit

<Window x:Class="ChudoObuv.OrderEditWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Style="{StaticResource AppWindow}"
        Title="Чудо Обувь — Редактирование заказа"
        Icon="/Resources/app.ico"
        Width="900" Height="640" MinWidth="760" MinHeight="500"
        WindowStartupLocation="CenterOwner">
    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <TextBlock x:Name="HeaderText" Grid.Row="0" Style="{StaticResource Heading}"/>

        <Grid Grid.Row="1" Margin="0,0,0,10">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="Auto"/>
                <ColumnDefinition Width="160"/>
                <ColumnDefinition Width="Auto"/>
                <ColumnDefinition Width="*"/>
            </Grid.ColumnDefinitions>
            <TextBlock Grid.Column="0" Text="Дата заказа:" FontWeight="SemiBold" VerticalAlignment="Center" Margin="0,0,8,0"/>
            <DatePicker x:Name="DateBox" Grid.Column="1"/>
            <TextBlock Grid.Column="2" Text="Клиент:" FontWeight="SemiBold" VerticalAlignment="Center" Margin="16,0,8,0"/>
            <ComboBox x:Name="CustomerBox" Grid.Column="3" DisplayMemberPath="Fio"/>
        </Grid>

        <DataGrid x:Name="LinesGrid" Grid.Row="2" AutoGenerateColumns="False" CanUserAddRows="False"
                  IsReadOnly="True" SelectionMode="Single" CanUserSortColumns="False">
            <DataGrid.Columns>
                <DataGridTextColumn Header="Товар" Binding="{Binding ProductName}" Width="*"/>
                <DataGridTextColumn Header="Размер" Binding="{Binding SizeText}" Width="80"/>
                <DataGridTextColumn Header="Цена" Binding="{Binding UnitPrice, StringFormat={}{0:N0} ₽}" Width="100"/>
                <DataGridTemplateColumn Header="Количество" Width="150">
                    <DataGridTemplateColumn.CellTemplate>
                        <DataTemplate>
                            <StackPanel Orientation="Horizontal">
                                <Button Content="−" Padding="9,1" Margin="2" Click="Minus_Click"/>
                                <TextBlock Text="{Binding Quantity}" Width="32" TextAlignment="Center"
                                           VerticalAlignment="Center" FontWeight="SemiBold"/>
                                <Button Content="+" Padding="9,1" Margin="2" Click="Plus_Click"/>
                            </StackPanel>
                        </DataTemplate>
                    </DataGridTemplateColumn.CellTemplate>
                </DataGridTemplateColumn>
                <DataGridTextColumn Header="Сумма" Binding="{Binding Sum, StringFormat={}{0:N0} ₽}" Width="110"/>
                <DataGridTemplateColumn Header="" Width="Auto">
                    <DataGridTemplateColumn.CellTemplate>
                        <DataTemplate>
                            <Button Content="Удалить" Padding="8,1" Margin="2" Click="Remove_Click"/>
                        </DataTemplate>
                    </DataGridTemplateColumn.CellTemplate>
                </DataGridTemplateColumn>
            </DataGrid.Columns>
        </DataGrid>

        <!-- Добавление новой товарной позиции -->
        <Border Grid.Row="3" Style="{StaticResource Panel}" Margin="0,10,0,0">
            <DockPanel>
                <Button DockPanel.Dock="Right" Content="Добавить позицию" Click="AddItem_Click"/>
                <ComboBox x:Name="StockBox" DisplayMemberPath="FullDisplay" Margin="0,0,8,0"/>
            </DockPanel>
        </Border>

        <DockPanel Grid.Row="4" Margin="0,12,0,0" LastChildFill="False">
            <TextBlock x:Name="TotalText" DockPanel.Dock="Left" FontSize="20" FontWeight="Bold"
                       VerticalAlignment="Center"/>
            <Button DockPanel.Dock="Right" Content="Отмена" IsCancel="True" Click="Cancel_Click"/>
            <Button DockPanel.Dock="Right" Content="Сохранить" Style="{StaticResource AccentButton}"
                    IsDefault="True" Click="Save_Click"/>
        </DockPanel>
    </Grid>
</Window>

OrderEdit.cs

using System.Collections.ObjectModel;
using System.ComponentModel;
using System.Windows;
using ChudoObuv.Models;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv;

/// <summary>Редактируемая копия строки заказа.</summary>
public class EditLine : INotifyPropertyChanged
{
    private int _quantity;

    public EditLine(StockItem item, int quantity, decimal unitPrice, OrderItem? existing)
    {
        Item = item;
        _quantity = quantity;
        UnitPrice = unitPrice;
        Existing = existing;
    }

    public StockItem Item { get; }
    public OrderItem? Existing { get; }
    public decimal UnitPrice { get; }

    public string ProductName => Item.Product.ProductName;
    public string SizeText => Item.SizeText;
    public decimal Sum => Quantity * UnitPrice;

    /// <summary>Сколько пар этой позиции уже входит в сохранённый заказ.</summary>
    public int OriginalQuantity => Existing?.Quantity ?? 0;

    /// <summary>Максимум, который можно указать: остаток на складе + то, что уже в заказе.</summary>
    public int MaxQuantity => Item.Quantity + OriginalQuantity;

    public int Quantity
    {
        get => _quantity;
        set
        {
            _quantity = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Quantity)));
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Sum)));
        }
    }

    public event PropertyChangedEventHandler? PropertyChanged;
}

/// <summary>Редактирование заказа (только администратор).</summary>
public partial class OrderEditWindow : Window
{
    private readonly Order _order;
    private readonly ObservableCollection<EditLine> _lines = new();

    public OrderEditWindow(Order order)
    {
        InitializeComponent();
        _order = order;

        HeaderText.Text = $"Редактирование заказа №{order.OrderId}";
        DateBox.SelectedDate = order.OrderDate.ToDateTime(TimeOnly.MinValue);

        var users = App.Db.Users.OrderBy(u => u.LastName).ThenBy(u => u.FirstName).ToList();
        CustomerBox.ItemsSource = users;
        CustomerBox.SelectedItem = users.FirstOrDefault(u => u.UserId == order.UserId);

        foreach (var item in order.OrderItems.OrderBy(i => i.OrderItemId))
            _lines.Add(new EditLine(item.StockItem, item.Quantity, item.UnitPrice, item));
        LinesGrid.ItemsSource = _lines;

        StockBox.ItemsSource = App.Db.StockItems
            .Include(s => s.Product)
            .ToList()
            .OrderBy(s => s.Product.ProductName)
            .ThenBy(s => s.SizeValue)
            .ToList();

        _lines.CollectionChanged += (_, _) => UpdateTotal();
        foreach (var line in _lines)
            line.PropertyChanged += (_, _) => UpdateTotal();
        UpdateTotal();
    }

    private void UpdateTotal()
    {
        TotalText.Text = $"Итого: {_lines.Sum(l => l.Sum):N0} ₽";
    }

    private void Plus_Click(object sender, RoutedEventArgs e)
    {
        if (((FrameworkElement)sender).DataContext is not EditLine line)
            return;

        if (line.Quantity + 1 > line.MaxQuantity)
        {
            MessageBox.Show($"Доступно для заказа только {line.MaxQuantity} пар.", "Чудо Обувь",
                MessageBoxButton.OK, MessageBoxImage.Warning);
            return;
        }

        line.Quantity++;
    }

    private void Minus_Click(object sender, RoutedEventArgs e)
    {
        if (((FrameworkElement)sender).DataContext is EditLine line && line.Quantity > 1)
            line.Quantity--;
    }

    private void Remove_Click(object sender, RoutedEventArgs e)
    {
        if (((FrameworkElement)sender).DataContext is EditLine line)
            _lines.Remove(line);
    }

    private void AddItem_Click(object sender, RoutedEventArgs e)
    {
        if (StockBox.SelectedItem is not StockItem item)
        {
            MessageBox.Show("Выберите товар и размер.", "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Information);
            return;
        }

        var line = _lines.FirstOrDefault(l => l.Item.StockItemId == item.StockItemId);
        if (line != null)
        {
            if (line.Quantity + 1 > line.MaxQuantity)
            {
                MessageBox.Show($"Доступно для заказа только {line.MaxQuantity} пар.", "Чудо Обувь",
                    MessageBoxButton.OK, MessageBoxImage.Warning);
                return;
            }

            line.Quantity++;
            return;
        }

        // позиция, ранее удалённая из заказа, но ещё не сохранённая, возвращается как существующая
        var existing = _order.OrderItems.FirstOrDefault(i => i.StockItemId == item.StockItemId);
        int max = item.Quantity + (existing?.Quantity ?? 0);
        if (max < 1)
        {
            MessageBox.Show("Этой позиции нет в наличии.", "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Warning);
            return;
        }

        var newLine = new EditLine(item, 1, existing?.UnitPrice ?? item.Product.Price, existing);
        newLine.PropertyChanged += (_, _) => UpdateTotal();
        _lines.Add(newLine);
    }

    private void Cancel_Click(object sender, RoutedEventArgs e)
    {
        DialogResult = false;
    }

    private void Save_Click(object sender, RoutedEventArgs e)
    {
        if (DateBox.SelectedDate == null)
        {
            MessageBox.Show("Укажите дату заказа.", "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Information);
            return;
        }

        if (CustomerBox.SelectedItem is not User customer)
        {
            MessageBox.Show("Выберите клиента.", "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Information);
            return;
        }

        if (_lines.Count == 0)
        {
            MessageBox.Show("Заказ должен содержать хотя бы одну позицию. Чтобы убрать заказ целиком, удалите его в списке заказов.",
                "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Information);
            return;
        }

        var lines = _lines.ToList();
        foreach (var line in lines)
        {
            if (line.Quantity > line.MaxQuantity)
            {
                MessageBox.Show($"«{line.ProductName}», размер {line.SizeText}: доступно только {line.MaxQuantity} пар.",
                    "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Warning);
                return;
            }
        }

        try
        {
            var db = App.Db;

            // 1. позиции, удалённые из заказа, возвращаются на склад
            foreach (var oi in _order.OrderItems.ToList())
            {
                if (lines.Any(l => l.Existing == oi))
                    continue;
                oi.StockItem.Quantity += oi.Quantity;
                db.OrderItems.Remove(oi);
            }

            // 2. изменённые и новые позиции
            foreach (var line in lines)
            {
                if (line.Existing != null)
                {
                    line.Item.Quantity -= line.Quantity - line.Existing.Quantity;
                    line.Existing.Quantity = line.Quantity;
                }
                else
                {
                    line.Item.Quantity -= line.Quantity;
                    _order.OrderItems.Add(new OrderItem
                    {
                        StockItemId = line.Item.StockItemId,
                        StockItem = line.Item,
                        Quantity = line.Quantity,
                        UnitPrice = line.UnitPrice
                    });
                }
            }

            _order.OrderDate = DateOnly.FromDateTime(DateBox.SelectedDate.Value);
            _order.User = customer;
            _order.UserId = customer.UserId;

            db.SaveChanges();
            DialogResult = true;
        }
        catch (Exception ex)
        {
            MessageBox.Show("Не удалось сохранить заказ:\n" + ex.GetBaseException().Message,
                "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Error);
            App.ResetDb();     // откатываем несохранённые изменения
            DialogResult = true;   // список заказов перечитает данные из БД
        }
    }
}

OrdersWindow

<Window x:Class="ChudoObuv.OrdersWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Style="{StaticResource AppWindow}"
        Title="Чудо Обувь — Заказы"
        Icon="/Resources/app.ico"
        Height="640" Width="1100" MinWidth="860" MinHeight="460"
        WindowStartupLocation="CenterOwner">
    <Grid Margin="16">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <TextBlock Grid.Row="0" Text="Заказы" Style="{StaticResource Heading}"/>

        <StackPanel Grid.Row="1" Orientation="Horizontal" Margin="0,0,0,10">
            <Button x:Name="NewButton" Content="Новый заказ" Style="{StaticResource AccentButton}"
                    Margin="0,4,4,4" Click="NewButton_Click"/>
            <Button x:Name="EditButton" Content="Редактировать" Click="EditButton_Click"/>
            <Button x:Name="DeleteButton" Content="Удалить" Click="DeleteButton_Click"/>
            <Button Content="Закрыть" Click="Close_Click"/>
        </StackPanel>

        <Grid Grid.Row="2">
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="1.3*"/>
                <ColumnDefinition Width="14"/>
                <ColumnDefinition Width="*"/>
            </Grid.ColumnDefinitions>

            <DataGrid x:Name="OrdersGrid" Grid.Column="0" AutoGenerateColumns="False" CanUserAddRows="False"
                      IsReadOnly="True" SelectionMode="Single" SelectionChanged="OrdersGrid_SelectionChanged">
                <DataGrid.Columns>
                    <DataGridTextColumn Header="№" Binding="{Binding OrderId}" Width="50"/>
                    <DataGridTextColumn Header="Дата заказа" Binding="{Binding OrderDate, StringFormat={}{0:dd.MM.yyyy}}" Width="110"/>
                    <DataGridTextColumn Header="ФИО клиента" Binding="{Binding CustomerFio}" Width="*"/>
                    <DataGridTextColumn Header="Пар" Binding="{Binding ItemsCount}" Width="50"/>
                    <DataGridTextColumn Header="Сумма" Binding="{Binding Total, StringFormat={}{0:N0} ₽}" Width="110"/>
                </DataGrid.Columns>
            </DataGrid>

            <Grid Grid.Column="2">
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                    <RowDefinition Height="Auto"/>
                </Grid.RowDefinitions>
                <TextBlock x:Name="DetailsTitle" Grid.Row="0" Text="Состав заказа" FontSize="18"
                           FontWeight="SemiBold" Margin="0,0,0,6"/>
                <DataGrid x:Name="ItemsGrid" Grid.Row="1" AutoGenerateColumns="False" CanUserAddRows="False"
                          IsReadOnly="True" SelectionMode="Single">
                    <DataGrid.Columns>
                        <DataGridTextColumn Header="Товар" Binding="{Binding ProductName}" Width="*"/>
                        <DataGridTextColumn Header="Разм." Binding="{Binding SizeText}" Width="55"/>
                        <DataGridTextColumn Header="Кол-во" Binding="{Binding Quantity}" Width="60"/>
                        <DataGridTextColumn Header="Цена" Binding="{Binding UnitPrice, StringFormat={}{0:N0} ₽}" Width="85"/>
                        <DataGridTextColumn Header="Сумма" Binding="{Binding Sum, StringFormat={}{0:N0} ₽}" Width="90"/>
                    </DataGrid.Columns>
                </DataGrid>
                <TextBlock x:Name="TotalText" Grid.Row="2" FontSize="18" FontWeight="Bold"
                           HorizontalAlignment="Right" Margin="0,8,0,0"/>
            </Grid>
        </Grid>
    </Grid>
</Window>

OrdersItem.cs

using System.Windows;
using System.Windows.Controls;
using ChudoObuv.Models;
using ChudoObuv.Services;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv
{
    /// <summary>Список заказов с составом выбранного заказа (менеджер и администратор).</summary>
    public partial class OrdersWindow : Window
    {
        public OrdersWindow()
        {
            InitializeComponent();
            EditButton.Visibility = Session.CanEditOrders ? Visibility.Visible : Visibility.Collapsed;
            LoadOrders();
        }

        private void LoadOrders(int? selectId = null)
        {
            try
            {
                var orders = App.Db.Orders
                    .Include(o => o.User)
                    .Include(o => o.OrderItems).ThenInclude(i => i.StockItem).ThenInclude(s => s.Product)
                    .OrderByDescending(o => o.OrderDate).ThenByDescending(o => o.OrderId)
                    .ToList();

                OrdersGrid.ItemsSource = orders;
                if (orders.Count > 0)
                    OrdersGrid.SelectedItem = orders.FirstOrDefault(o => o.OrderId == selectId) ?? orders[0];
                else
                    ShowDetails(null);
            }
            catch (Exception ex)
            {
                MessageBox.Show("Не удалось загрузить заказы:\n" + ex.GetBaseException().Message,
                    "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Error);
            }
        }

        private void OrdersGrid_SelectionChanged(object sender, SelectionChangedEventArgs e)
            => ShowDetails(OrdersGrid.SelectedItem as Order);

        private void ShowDetails(Order? order)
        {
            if (order == null)
            {
                DetailsTitle.Text = "Состав заказа";
                ItemsGrid.ItemsSource = null;
                TotalText.Text = "";
                return;
            }

            DetailsTitle.Text = $"Состав заказа №{order.OrderId}";
            ItemsGrid.ItemsSource = order.OrderItems.OrderBy(i => i.OrderItemId).ToList();
            TotalText.Text = $"Итого: {order.Total:N0} ₽";
        }

        private void NewButton_Click(object sender, RoutedEventArgs e)
        {
            MessageBox.Show(
                "Выберите в каталоге нужные товары и размеры, нажав «Добавить в заказ», " +
                "затем откройте «Мой заказ», укажите клиента и оформите заказ.",
                "Новый заказ", MessageBoxButton.OK, MessageBoxImage.Information);
            Close();
        }

        private void EditButton_Click(object sender, RoutedEventArgs e)
        {
            if (OrdersGrid.SelectedItem is not Order order)
            {
                MessageBox.Show("Выберите заказ.", "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Information);
                return;
            }

            var window = new OrderEditWindow(order) { Owner = this };
            if (window.ShowDialog() == true)
                LoadOrders(order.OrderId);
        }

        private void DeleteButton_Click(object sender, RoutedEventArgs e)
        {
            if (OrdersGrid.SelectedItem is not Order order)
            {
                MessageBox.Show("Выберите заказ.", "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Information);
                return;
            }

            var answer = MessageBox.Show(
                $"Удалить заказ №{order.OrderId} ({order.CustomerFio})?\nТовары из заказа вернутся на склад.",
                "Удаление заказа", MessageBoxButton.YesNo, MessageBoxImage.Question);
            if (answer != MessageBoxResult.Yes)
                return;

            try
            {
                var db = App.Db;
                foreach (var item in order.OrderItems.ToList())
                {
                    item.StockItem.Quantity += item.Quantity;   // возвращаем на склад
                    db.OrderItems.Remove(item);
                }

                db.Orders.Remove(order);
                db.SaveChanges();
                LoadOrders();
            }
            catch (Exception ex)
            {
                MessageBox.Show("Не удалось удалить заказ:\n" + ex.GetBaseException().Message,
                    "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Error);
                App.ResetDb();   // сбрасываем несохранённые изменения
                LoadOrders();
            }
        }

        private void Close_Click(object sender, RoutedEventArgs e) => Close();
    }
}


Products

<Window x:Class="ChudoObuv.Products"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Style="{StaticResource AppWindow}"
        Title="Чудо Обувь — Каталог товаров"
        Icon="/Resources/app.ico"
        Height="800" Width="1280" MinWidth="900" MinHeight="560"
        WindowStartupLocation="CenterScreen">
    <DockPanel>

        <!-- Шапка с логотипом -->
        <Border DockPanel.Dock="Top" Background="{StaticResource AltBg}"
                BorderBrush="{StaticResource Accent}" BorderThickness="0,0,0,1" Padding="14,6">
            <DockPanel LastChildFill="False">
                <Image DockPanel.Dock="Left" Source="/Resources/logo.png" Height="64" Stretch="Uniform"/>
                <StackPanel DockPanel.Dock="Left" VerticalAlignment="Center" Margin="10,0,24,0">
                    <TextBlock Text="Чудо Обувь" FontSize="26" FontWeight="Bold"/>
                    <TextBlock Text="Каталог товаров" Foreground="{StaticResource MutedBrush}"/>
                </StackPanel>

                <Button DockPanel.Dock="Right" Content="Выйти" Style="{StaticResource LightButton}"
                        VerticalAlignment="Center" Click="Logout_Click"/>
                <Button DockPanel.Dock="Right" x:Name="CartButton" Content="Мой заказ (0)"
                        Style="{StaticResource AccentButton}" VerticalAlignment="Center" Click="Cart_Click"/>
                <Button DockPanel.Dock="Right" x:Name="OrdersButton" Content="Заказы"
                        Style="{StaticResource LightButton}" VerticalAlignment="Center" Click="Orders_Click"/>
                <StackPanel DockPanel.Dock="Right" VerticalAlignment="Center" Margin="0,0,14,0">
                    <TextBlock x:Name="UserNameText" FontWeight="SemiBold" HorizontalAlignment="Right"/>
                    <TextBlock x:Name="UserRoleText" HorizontalAlignment="Right" FontSize="12"
                               Foreground="{StaticResource MutedBrush}"/>
                </StackPanel>
            </DockPanel>
        </Border>

        <!-- Поиск, фильтрация и сортировка -->
        <Border DockPanel.Dock="Top" Style="{StaticResource Panel}" Margin="10,10,10,0">
            <Grid>
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="2*"/>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="*"/>
                </Grid.ColumnDefinitions>

                <StackPanel Grid.Column="0" Margin="0,0,10,0">
                    <TextBlock Text="Поиск" Style="{StaticResource FieldLabel}"/>
                    <TextBox x:Name="SearchBox" TextChanged="Filter_Changed"/>
                </StackPanel>
                <StackPanel Grid.Column="1" Margin="0,0,10,0">
                    <TextBlock Text="Категория" Style="{StaticResource FieldLabel}"/>
                    <ComboBox x:Name="CategoryBox" SelectionChanged="Filter_Changed"/>
                </StackPanel>
                <StackPanel Grid.Column="2" Margin="0,0,10,0">
                    <TextBlock Text="Производитель" Style="{StaticResource FieldLabel}"/>
                    <ComboBox x:Name="ManufacturerBox" SelectionChanged="Filter_Changed"/>
                </StackPanel>
                <StackPanel Grid.Column="3">
                    <TextBlock Text="Сортировка" Style="{StaticResource FieldLabel}"/>
                    <ComboBox x:Name="SortBox" SelectionChanged="Filter_Changed"/>
                </StackPanel>
            </Grid>
        </Border>

        <Grid>
            <ScrollViewer VerticalScrollBarVisibility="Auto">
                <!-- Каждый товар выводится через UserControl ProductUserControl -->
                <WrapPanel x:Name="panel" Margin="10"/>
            </ScrollViewer>
            <TextBlock x:Name="EmptyText" Text="Товары не найдены." FontSize="16"
                       Foreground="{StaticResource MutedBrush}"
                       HorizontalAlignment="Center" VerticalAlignment="Center" Visibility="Collapsed"/>
        </Grid>
    </DockPanel>
</Window>

Products

using System.Windows;
using System.Windows.Controls;
using ChudoObuv.Models;
using ChudoObuv.Services;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv
{
    /// <summary>Каталог для авторизованного пользователя, менеджера и администратора.</summary>
    public partial class Products : Window
    {
        private const string AllCategories = "Все категории";
        private const string AllManufacturers = "Все производители";

        private static readonly string[] SortModes =
        {
            "Без сортировки",
            "Цена: по возрастанию",
            "Цена: по убыванию",
            "Название: А — Я",
            "Название: Я — А",
            "Остаток: по возрастанию",
            "Остаток: по убыванию"
        };

        private List<Product> _all = new();
        private bool _ready;

        public Products(User user)
        {
            InitializeComponent();

            UserNameText.Text = user.Fio;
            UserRoleText.Text = user.Role.RoleName;
            OrdersButton.Visibility = Session.CanManageOrders ? Visibility.Visible : Visibility.Collapsed;

            Cart.Changed += UpdateCartText;
            Closed += (_, _) => Cart.Changed -= UpdateCartText;
            UpdateCartText();

            LoadData();
        }

        private void UpdateCartText()
        {
            CartButton.Content = $"Мой заказ ({Cart.TotalCount})";
        }

        public void LoadData()
        {
            try
            {
                _all = App.Db.Products
                    .Include(p => p.Manufacturer)
                    .Include(p => p.Subcategory).ThenInclude(s => s.Category)
                    .Include(p => p.StockItems)
                    .OrderBy(p => p.ProductId)
                    .ToList();
            }
            catch (Exception ex)
            {
                MessageBox.Show("Не удалось загрузить товары:\n" + ex.GetBaseException().Message,
                    "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Error);
                return;
            }

            if (!_ready)
            {
                CategoryBox.ItemsSource = new[] { AllCategories }
                    .Concat(_all.Select(p => p.CategoryName).Distinct().OrderBy(n => n)).ToList();
                ManufacturerBox.ItemsSource = new[] { AllManufacturers }
                    .Concat(_all.Select(p => p.Manufacturer.ManufacturerName).Distinct().OrderBy(n => n)).ToList();
                SortBox.ItemsSource = SortModes;
                CategoryBox.SelectedIndex = 0;
                ManufacturerBox.SelectedIndex = 0;
                SortBox.SelectedIndex = 0;
                _ready = true;
            }

            Apply();
        }

        private void Filter_Changed(object sender, RoutedEventArgs e)
        {
            if (_ready)
                Apply();
        }

        private void Apply()
        {
            IEnumerable<Product> items = _all;

            var text = SearchBox.Text.Trim();
            if (text.Length > 0)
            {
                items = items.Where(p =>
                    Contains(p.ProductName, text) ||
                    Contains(p.Description, text) ||
                    Contains(p.Composition, text) ||
                    Contains(p.Manufacturer.ManufacturerName, text) ||
                    Contains(p.Subcategory.SubcategoryName, text) ||
                    Contains(p.CategoryName, text));
            }

            if (CategoryBox.SelectedItem is string cat && cat != AllCategories)
                items = items.Where(p => p.CategoryName == cat);

            if (ManufacturerBox.SelectedItem is string man && man != AllManufacturers)
                items = items.Where(p => p.Manufacturer.ManufacturerName == man);

            items = SortBox.SelectedIndex switch
            {
                1 => items.OrderBy(p => p.Price),
                2 => items.OrderByDescending(p => p.Price),
                3 => items.OrderBy(p => p.ProductName, StringComparer.CurrentCultureIgnoreCase),
                4 => items.OrderByDescending(p => p.ProductName, StringComparer.CurrentCultureIgnoreCase),
                5 => items.OrderBy(p => p.TotalQuantity),
                6 => items.OrderByDescending(p => p.TotalQuantity),
                _ => items
            };

            panel.Children.Clear();
            int count = 0;
            foreach (var item in items)
            {
                var control = new ProductUserControl();
                control.LoadData(item);
                panel.Children.Add(control);
                count++;
            }

            EmptyText.Visibility = count == 0 ? Visibility.Visible : Visibility.Collapsed;
        }

        private static bool Contains(string? source, string text)
            => source != null && source.Contains(text, StringComparison.CurrentCultureIgnoreCase);

        private void Cart_Click(object sender, RoutedEventArgs e)
        {
            var window = new CartWindow { Owner = this };
            window.ShowDialog();
            if (window.OrderCreated)
                LoadData();      // остатки изменились
        }

        private void Orders_Click(object sender, RoutedEventArgs e)
        {
            new OrdersWindow { Owner = this }.ShowDialog();
            LoadData();          // заказы могли быть изменены или удалены
        }

        private void Logout_Click(object sender, RoutedEventArgs e)
        {
            Cart.Clear();
            Session.Logout();
            App.ResetDb();
            new Auth().Show();
            Close();
        }
    }
}

ProductUserControl

<UserControl x:Class="ChudoObuv.ProductUserControl"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             mc:Ignorable="d"
             d:DesignHeight="200" d:DesignWidth="580">
    <!-- Карточка товара. Заполняется методом LoadData(Product). -->
    <Border x:Name="Card" Style="{StaticResource Card}" Width="580" Margin="6">
        <Grid>
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="136"/>
                <ColumnDefinition Width="*"/>
            </Grid.ColumnDefinitions>

            <Border Grid.Column="0" Background="White" BorderBrush="{StaticResource AltBg}" BorderThickness="1"
                    CornerRadius="{StaticResource Radius}" Width="124" Height="124"
                    VerticalAlignment="Top" HorizontalAlignment="Left">
                <Image x:Name="Photo" Stretch="Uniform" Margin="4"/>
            </Border>

            <StackPanel Grid.Column="1">
                <TextBlock x:Name="tbTitle" FontSize="17" FontWeight="Bold" TextWrapping="Wrap"/>
                <TextBlock x:Name="tbCategory" Foreground="{StaticResource MutedBrush}" Margin="0,2,0,0"/>
                <TextBlock x:Name="tbManufacturer" TextWrapping="Wrap" Margin="0,4,0,0"/>
                <TextBlock x:Name="tbComposition" TextWrapping="Wrap" TextTrimming="CharacterEllipsis"
                           MaxHeight="40" Margin="0,2,0,0"/>
                <TextBlock x:Name="tbSizes" TextWrapping="Wrap" Margin="0,2,0,0"/>

                <Grid Margin="0,8,0,0">
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="*"/>
                        <ColumnDefinition Width="Auto"/>
                    </Grid.ColumnDefinitions>
                    <TextBlock x:Name="tbCount" VerticalAlignment="Center" FontWeight="SemiBold"/>
                    <TextBlock x:Name="tbPrice" Grid.Column="1" FontSize="22" FontWeight="Bold"/>
                </Grid>

                <!-- Блок заказа виден только авторизованным пользователям -->
                <Grid x:Name="OrderPanel" Margin="0,8,0,0" Visibility="Collapsed">
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="*"/>
                        <ColumnDefinition Width="Auto"/>
                    </Grid.ColumnDefinitions>
                    <ComboBox x:Name="SizeBox" DisplayMemberPath="SizeDisplay"/>
                    <Button x:Name="AddButton" Grid.Column="1" Content="Добавить в заказ"
                            Style="{StaticResource AccentButton}" Margin="8,0,0,0" Click="AddButton_Click"/>
                </Grid>
                <TextBlock x:Name="AddedText" HorizontalAlignment="Right" FontSize="12"
                           Foreground="{StaticResource AccentDark}" Margin="0,3,0,0" Visibility="Collapsed"/>
            </StackPanel>
        </Grid>
    </Border>
</UserControl>

ProductUserControl.cs

using System.IO;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using ChudoObuv.Models;
using ChudoObuv.Services;

namespace ChudoObuv
{
    /// <summary>Карточка товара для списка каталога.</summary>
    public partial class ProductUserControl : UserControl
    {
        private static readonly Dictionary<string, BitmapImage?> ImageCache = new();

        public ProductUserControl()
        {
            InitializeComponent();
        }

        public void LoadData(Product product)
        {
            tbTitle.Text = product.ProductName;
            tbCategory.Text = product.CategoryText;
            tbManufacturer.Text = "Производитель: " + product.Manufacturer.ManufacturerName;
            tbComposition.Text = "Состав: " + product.Composition;
            tbComposition.ToolTip = product.Description;
            tbSizes.Text = "Размеры: " + product.SizesText;
            tbCount.Text = $"В наличии: {product.TotalQuantity} пар";
            tbPrice.Text = product.PriceText;
            Photo.Source = LoadImage(product.ImagePath);

            // товары с суммарным количеством <= 3 подсвечиваются
            if (product.IsLowStock)
                Card.Background = (Brush)Application.Current.FindResource("LowStockBg");

            if (Session.CanOrder)
            {
                var available = product.AvailableStock;
                if (available.Count > 0)
                {
                    SizeBox.ItemsSource = available;
                    SizeBox.SelectedIndex = 0;
                    OrderPanel.Visibility = Visibility.Visible;
                }
            }
        }

        private void AddButton_Click(object sender, RoutedEventArgs e)
        {
            if (SizeBox.SelectedItem is not StockItem item)
            {
                MessageBox.Show("Выберите размер.", "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Information);
                return;
            }

            if (!Cart.TryAdd(item, out var message))
            {
                MessageBox.Show(message, "Чудо Обувь", MessageBoxButton.OK, MessageBoxImage.Warning);
                return;
            }

            AddedText.Text = "Добавлено в заказ";
            AddedText.Visibility = Visibility.Visible;
        }

        private static BitmapImage? LoadImage(string path)
        {
            if (ImageCache.TryGetValue(path, out var cached))
                return cached;

            BitmapImage? image = null;
            try
            {
                var file = File.Exists(path)
                    ? path
                    : Path.Combine(AppContext.BaseDirectory, "Images", "picture.png");
                if (File.Exists(file))
                {
                    image = new BitmapImage();
                    image.BeginInit();
                    image.CacheOption = BitmapCacheOption.OnLoad;
                    image.DecodePixelWidth = 240;
                    image.UriSource = new Uri(file, UriKind.Absolute);
                    image.EndInit();
                    image.Freeze();
                }
            }
            catch
            {
                image = null;
            }

            ImageCache[path] = image;
            return image;
        }
    }
}


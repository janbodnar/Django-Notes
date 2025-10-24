# Management commands

Django management commands are custom scripts that can be run from the command line using `manage.py`.  
They are useful for automating tasks, data migration, maintenance, and administrative operations.

First, we need to manually create `management/commands` directories inside an app.  
The command file should be a Python file in the `management/commands` directory.

Run `py manage.py` to list all available commands.  

## Basic command

The filename is `hello.py`:

```python
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = 'Displays hello message'

    def handle(self, *args, **kwargs):
        self.stdout.write(self.style.NOTICE('Hello!'))
```

The command is then launched with `py manage.py hello`. 

## Command with colored output

Django provides several output styles for command messages.

```python
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = 'Demonstrates different output styles'

    def handle(self, *args, **kwargs):
        self.stdout.write(self.style.SUCCESS('This is a success message'))
        self.stdout.write(self.style.WARNING('This is a warning message'))
        self.stdout.write(self.style.ERROR('This is an error message'))
        self.stdout.write(self.style.NOTICE('This is a notice message'))
        self.stdout.write(self.style.HTTP_INFO('This is an HTTP info message'))
        self.stdout.write(self.style.HTTP_SUCCESS('This is an HTTP success message'))
```

## Command with arguments and options

Commands can accept both positional arguments and named options.

```python
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = 'Process user data with options'

    def add_arguments(self, parser):
        # Positional argument
        parser.add_argument('username', type=str, help='Username to process')
        
        # Named optional argument
        parser.add_argument(
            '--email',
            type=str,
            help='User email address',
        )
        
        # Flag option
        parser.add_argument(
            '--active',
            action='store_true',
            help='Mark user as active',
        )

    def handle(self, *args, **kwargs):
        username = kwargs['username']
        email = kwargs.get('email')
        is_active = kwargs['active']
        
        self.stdout.write(f'Processing user: {username}')
        if email:
            self.stdout.write(f'Email: {email}')
        if is_active:
            self.stdout.write(self.style.SUCCESS('User marked as active'))
```

Usage: `py manage.py process_user john --email john@example.com --active`

## Seed command 

`py manage.py flush` cleans the database; need to create a new superuser.  

Model definition in `models.py`:

```python
from django.db import models

class Message(models.Model):
    text = models.CharField(max_length=255)
    created = models.DateField(auto_now_add=False)
```

Command to seed database with fake data in `management/commands/seed.py`:

```python
from django.core.management.base import BaseCommand
from faker import Faker

from ...models import Message


class Command(BaseCommand):
    help = 'Seed database with fake message data'

    def add_arguments(self, parser):
        group = parser.add_mutually_exclusive_group()
        
        group.add_argument(
            'total', 
            type=int, 
            default=10,
            help='Number of messages to create', 
            nargs='?'
        )
        
        group.add_argument(
            '-c', '--clear', 
            action='store_true', 
            help='Clear all messages'
        )

    def handle(self, *args, **kwargs):
        clear_flag = kwargs['clear']

        if clear_flag:
            Message.objects.all().delete()
            self.stdout.write(self.style.WARNING('All messages deleted'))
            return

        total = kwargs['total']
        faker = Faker()

        for _ in range(total):
            msg = faker.text(max_nb_chars=30)
            created = faker.date_between('-10y', 'today')
            message = Message(text=msg, created=created)
            message.save()

        self.stdout.write(self.style.SUCCESS(f'{total} messages generated'))
```

Usage: `py manage.py seed 50` or `py manage.py seed --clear`

## Export data to CSV

Export model data to a CSV file for backup or analysis.

```python
import csv
from django.core.management.base import BaseCommand
from django.apps import apps

class Command(BaseCommand):
    help = 'Export model data to CSV file'

    def add_arguments(self, parser):
        parser.add_argument('model', type=str, help='Model name (app.Model)')
        parser.add_argument('--output', type=str, default='export.csv', help='Output CSV file')

    def handle(self, *args, **kwargs):
        model_name = kwargs['model']
        output_file = kwargs['output']
        
        try:
            app_label, model = model_name.split('.')
            Model = apps.get_model(app_label, model)
        except (ValueError, LookupError):
            self.stdout.write(self.style.ERROR(f'Invalid model: {model_name}'))
            return
        
        queryset = Model.objects.all()
        
        if not queryset.exists():
            self.stdout.write(self.style.WARNING('No data to export'))
            return
        
        field_names = [field.name for field in Model._meta.fields]
        
        with open(output_file, 'w', newline='', encoding='utf-8') as csvfile:
            writer = csv.writer(csvfile)
            writer.writerow(field_names)
            
            for obj in queryset:
                writer.writerow([getattr(obj, field) for field in field_names])
        
        self.stdout.write(self.style.SUCCESS(f'Exported {queryset.count()} records to {output_file}'))
```

Usage: `py manage.py export_csv myapp.Message --output messages.csv`

## Import data from JSON

Import data from a JSON file into the database.

```python
import json
from django.core.management.base import BaseCommand
from django.apps import apps

class Command(BaseCommand):
    help = 'Import data from JSON file'

    def add_arguments(self, parser):
        parser.add_argument('json_file', type=str, help='Path to JSON file')
        parser.add_argument('model', type=str, help='Model name (app.Model)')

    def handle(self, *args, **kwargs):
        json_file = kwargs['json_file']
        model_name = kwargs['model']
        
        try:
            app_label, model = model_name.split('.')
            Model = apps.get_model(app_label, model)
        except (ValueError, LookupError):
            self.stdout.write(self.style.ERROR(f'Invalid model: {model_name}'))
            return
        
        try:
            with open(json_file, 'r', encoding='utf-8') as f:
                data = json.load(f)
        except FileNotFoundError:
            self.stdout.write(self.style.ERROR(f'File not found: {json_file}'))
            return
        except json.JSONDecodeError:
            self.stdout.write(self.style.ERROR('Invalid JSON file'))
            return
        
        created_count = 0
        for item in data:
            Model.objects.create(**item)
            created_count += 1
        
        self.stdout.write(self.style.SUCCESS(f'Imported {created_count} records'))
```

Usage: `py manage.py import_json data.json myapp.Message`

## Delete old records

Clean up old database records based on a date field.

```python
from django.core.management.base import BaseCommand
from django.utils import timezone
from datetime import timedelta

from ...models import Message

class Command(BaseCommand):
    help = 'Delete old message records'

    def add_arguments(self, parser):
        parser.add_argument(
            '--days',
            type=int,
            default=30,
            help='Delete records older than specified days (default: 30)',
        )
        
        parser.add_argument(
            '--dry-run',
            action='store_true',
            help='Show what would be deleted without actually deleting',
        )

    def handle(self, *args, **kwargs):
        days = kwargs['days']
        dry_run = kwargs['dry_run']
        
        cutoff_date = timezone.now().date() - timedelta(days=days)
        old_records = Message.objects.filter(created__lt=cutoff_date)
        count = old_records.count()
        
        if dry_run:
            self.stdout.write(
                self.style.WARNING(f'DRY RUN: Would delete {count} records older than {cutoff_date}')
            )
        else:
            old_records.delete()
            self.stdout.write(
                self.style.SUCCESS(f'Deleted {count} records older than {cutoff_date}')
            )
```

Usage: `py manage.py cleanup_old --days 60` or `py manage.py cleanup_old --days 60 --dry-run`

## Send email notifications

Send bulk email notifications to users.

```python
from django.core.management.base import BaseCommand
from django.core.mail import send_mail
from django.contrib.auth.models import User
from django.conf import settings

class Command(BaseCommand):
    help = 'Send email notification to all active users'

    def add_arguments(self, parser):
        parser.add_argument('--subject', type=str, required=True, help='Email subject')
        parser.add_argument('--message', type=str, required=True, help='Email message')

    def handle(self, *args, **kwargs):
        subject = kwargs['subject']
        message = kwargs['message']
        
        active_users = User.objects.filter(is_active=True, email__isnull=False).exclude(email='')
        
        sent_count = 0
        for user in active_users:
            try:
                send_mail(
                    subject,
                    message,
                    settings.DEFAULT_FROM_EMAIL,
                    [user.email],
                    fail_silently=False,
                )
                sent_count += 1
                self.stdout.write(f'Sent to {user.email}')
            except Exception as e:
                self.stdout.write(self.style.ERROR(f'Failed to send to {user.email}: {str(e)}'))
        
        self.stdout.write(self.style.SUCCESS(f'Successfully sent {sent_count} emails'))
```

Usage: `py manage.py send_notifications --subject "Update" --message "System maintenance tonight"`

## Generate reports

Generate a summary report of database statistics.

```python
from django.core.management.base import BaseCommand
from django.contrib.auth.models import User
from django.utils import timezone

from ...models import Message

class Command(BaseCommand):
    help = 'Generate database statistics report'

    def handle(self, *args, **kwargs):
        self.stdout.write(self.style.HTTP_INFO('=' * 50))
        self.stdout.write(self.style.HTTP_INFO('DATABASE STATISTICS REPORT'))
        self.stdout.write(self.style.HTTP_INFO('=' * 50))
        
        # User statistics
        total_users = User.objects.count()
        active_users = User.objects.filter(is_active=True).count()
        staff_users = User.objects.filter(is_staff=True).count()
        
        self.stdout.write('\nUser Statistics:')
        self.stdout.write(f'  Total users: {total_users}')
        self.stdout.write(f'  Active users: {active_users}')
        self.stdout.write(f'  Staff users: {staff_users}')
        
        # Message statistics
        total_messages = Message.objects.count()
        today = timezone.now().date()
        recent_messages = Message.objects.filter(created=today).count()
        
        self.stdout.write('\nMessage Statistics:')
        self.stdout.write(f'  Total messages: {total_messages}')
        self.stdout.write(f'  Messages today: {recent_messages}')
        
        self.stdout.write(self.style.HTTP_INFO('\n' + '=' * 50))
        self.stdout.write(self.style.SUCCESS('Report generated successfully'))
```

Usage: `py manage.py generate_report`

## Command with progress bar

Display progress for long-running operations.

```python
from django.core.management.base import BaseCommand
from django.db.models import Q
import time

from ...models import Message

class Command(BaseCommand):
    help = 'Process messages with progress indicator'

    def handle(self, *args, **kwargs):
        messages = Message.objects.all()
        total = messages.count()
        
        self.stdout.write(f'Processing {total} messages...')
        
        processed = 0
        for message in messages:
            # Simulate processing
            # In real scenario, perform actual data processing here
            time.sleep(0.01)
            
            processed += 1
            progress = int((processed / total) * 100)
            
            # Display progress
            self.stdout.write(f'\rProgress: {progress}% ({processed}/{total})', ending='')
            self.stdout.flush()
        
        self.stdout.write('')  # New line
        self.stdout.write(self.style.SUCCESS(f'Successfully processed {processed} messages'))
```

Usage: `py manage.py process_messages`

## Data validation command

Validate data integrity and find inconsistencies.

```python
from django.core.management.base import BaseCommand
from django.contrib.auth.models import User

from ...models import Message

class Command(BaseCommand):
    help = 'Validate data integrity'

    def handle(self, *args, **kwargs):
        issues_found = 0
        
        self.stdout.write(self.style.HTTP_INFO('Starting data validation...'))
        
        # Check for users without email
        users_no_email = User.objects.filter(Q(email='') | Q(email__isnull=True))
        if users_no_email.exists():
            count = users_no_email.count()
            self.stdout.write(self.style.WARNING(f'Found {count} users without email'))
            issues_found += count
        
        # Check for empty messages
        empty_messages = Message.objects.filter(Q(text='') | Q(text__isnull=True))
        if empty_messages.exists():
            count = empty_messages.count()
            self.stdout.write(self.style.WARNING(f'Found {count} empty messages'))
            issues_found += count
        
        # Check for duplicate messages
        from django.db.models import Count
        duplicates = Message.objects.values('text').annotate(
            count=Count('text')
        ).filter(count__gt=1)
        
        if duplicates.exists():
            count = duplicates.count()
            self.stdout.write(self.style.WARNING(f'Found {count} duplicate message texts'))
            issues_found += count
        
        if issues_found == 0:
            self.stdout.write(self.style.SUCCESS('No data issues found!'))
        else:
            self.stdout.write(self.style.ERROR(f'Total issues found: {issues_found}'))
```

Usage: `py manage.py validate_data`

## Scheduled task command

Command designed to be run as a cron job for periodic tasks.

```python
from django.core.management.base import BaseCommand
from django.utils import timezone
from django.core.mail import mail_admins
import logging

from ...models import Message

logger = logging.getLogger(__name__)

class Command(BaseCommand):
    help = 'Daily maintenance tasks (designed for cron)'

    def add_arguments(self, parser):
        parser.add_argument(
            '--send-report',
            action='store_true',
            help='Send email report to admins',
        )

    def handle(self, *args, **kwargs):
        send_report = kwargs['send_report']
        
        timestamp = timezone.now()
        self.stdout.write(f'Running daily tasks at {timestamp}')
        
        # Task 1: Clean up old data
        old_date = timestamp.date() - timezone.timedelta(days=90)
        deleted_count = Message.objects.filter(created__lt=old_date).delete()[0]
        
        # Task 2: Generate statistics
        total_messages = Message.objects.count()
        today_messages = Message.objects.filter(created=timestamp.date()).count()
        
        report = f'''
        Daily Maintenance Report - {timestamp.date()}
        
        Tasks Completed:
        - Deleted {deleted_count} old messages (>90 days)
        - Total messages in database: {total_messages}
        - Messages created today: {today_messages}
        '''
        
        self.stdout.write(report)
        logger.info(report)
        
        if send_report:
            mail_admins(
                subject=f'Daily Maintenance Report - {timestamp.date()}',
                message=report,
                fail_silently=True,
            )
            self.stdout.write(self.style.SUCCESS('Report sent to admins'))
        
        self.stdout.write(self.style.SUCCESS('Daily tasks completed successfully'))
```

Usage: `py manage.py daily_tasks --send-report`

Add to crontab: `0 2 * * * cd /path/to/project && /path/to/python manage.py daily_tasks --send-report`
